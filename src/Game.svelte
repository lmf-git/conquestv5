<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import RAPIER from '@dimforge/rapier3d-compat';
  
  let canvas = $state();
  let exteriorScene, interiorScene, stationScene, exteriorCamera, interiorCamera, stationCamera, renderer;
  let exteriorWorld, interiorWorld, stationWorld, exteriorEventQueue, interiorEventQueue, stationEventQueue;
  let playerBody, playerCollider;
  let interiorPlayerBody, interiorPlayerCollider;
  let stationPlayerBody, stationPlayerCollider;
  let shipBody, shipCollider;
  let stationShipBody, stationShipCollider; // Ship physics inside station
  let stationBody, stationCollider;
  let isInsideShip = $state(false);
  let isInsideStation = $state(false);
  let isShipDocked = $state(false);
  // Ship docking is now immediate like player entry - no transition states needed
  let shipInteriorProxy, stationInteriorProxy;
  let transitionProgress = $state(0); // 0 = fully outside, 1 = fully inside
  let isTransitioning = $state(false);
  let isExiting = $state(false);
  let isEntering = $state(false);
  let pendingShipEntry = $state(false);
  let pendingStationEntry = $state(false);
  let pendingShipDocking = $state(false);
  let lastUndockTime = $state(0); // Cooldown to prevent immediate re-docking
  let lastShipExitToStationTime = $state(0); // Cooldown to prevent station re-entry after ship exit
  let pendingShipEntryFromStation = $state(false); // Deferred ship entry from station
  let isFirstPerson = $state(false);
  let isPointerLocked = $state(false);
  let mouseX = 0;
  let mouseY = 0;
  let cameraRotation = { yaw: 0, pitch: 0 };
  
  onMount(async () => {
    await RAPIER.init();
    
    // Create separate scenes for exterior, ship interior, and station interior
    exteriorScene = new THREE.Scene();
    exteriorScene.background = new THREE.Color(0x87CEEB);
    exteriorScene.fog = new THREE.Fog(0x87CEEB, 10, 1000);
    
    interiorScene = new THREE.Scene();
    interiorScene.background = new THREE.Color(0x222222); // Dark ship interior background
    
    stationScene = new THREE.Scene();
    stationScene.background = new THREE.Color(0x333333); // Station interior background
    
    exteriorCamera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    exteriorCamera.position.set(0, 5, 10);
    
    interiorCamera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    interiorCamera.position.set(0, 4, 12);
    
    stationCamera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
    stationCamera.position.set(0, 40, 100);
    
    renderer = new THREE.WebGLRenderer({ canvas, antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.shadowMap.enabled = true;
    
    // Exterior scene lighting
    const exteriorAmbientLight = new THREE.AmbientLight(0xffffff, 0.6);
    exteriorScene.add(exteriorAmbientLight);
    
    const exteriorDirectionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
    exteriorDirectionalLight.position.set(10, 20, 10);
    exteriorDirectionalLight.castShadow = true;
    exteriorDirectionalLight.shadow.camera.near = 0.1;
    exteriorDirectionalLight.shadow.camera.far = 100;
    exteriorDirectionalLight.shadow.camera.left = -50;
    exteriorDirectionalLight.shadow.camera.right = 50;
    exteriorDirectionalLight.shadow.camera.top = 50;
    exteriorDirectionalLight.shadow.camera.bottom = -50;
    exteriorScene.add(exteriorDirectionalLight);
    
    // Interior scene lighting
    const interiorAmbientLight = new THREE.AmbientLight(0xffffff, 0.4);
    interiorScene.add(interiorAmbientLight);
    
    // Station scene lighting
    const stationAmbientLight = new THREE.AmbientLight(0xffffff, 0.5);
    stationScene.add(stationAmbientLight);
    
    const stationDirectionalLight = new THREE.DirectionalLight(0xffffff, 0.6);
    stationDirectionalLight.position.set(5, 10, 5);
    stationScene.add(stationDirectionalLight);
    
    const gravity = { x: 0.0, y: -9.81, z: 0.0 };
    exteriorWorld = new RAPIER.World(gravity);
    exteriorEventQueue = new RAPIER.EventQueue(true);
    
    interiorWorld = new RAPIER.World(gravity);
    interiorEventQueue = new RAPIER.EventQueue(true);
    
    stationWorld = new RAPIER.World(gravity);
    stationEventQueue = new RAPIER.EventQueue(true);
    
    createGround();
    createPlayer();
    createShip();
    createDockingStation();
    
    window.addEventListener('resize', onWindowResize);
    document.addEventListener('keydown', handleKeyDown);
    document.addEventListener('keyup', handleKeyUp);
    document.addEventListener('mousemove', handleMouseMove);
    document.addEventListener('click', handleClick);
    document.addEventListener('pointerlockchange', handlePointerLockChange);
    
    animate();
    
    return () => {
      window.removeEventListener('resize', onWindowResize);
      document.removeEventListener('keydown', handleKeyDown);
      document.removeEventListener('keyup', handleKeyUp);
      document.removeEventListener('mousemove', handleMouseMove);
      document.removeEventListener('click', handleClick);
      document.removeEventListener('pointerlockchange', handlePointerLockChange);
    };
  });
  
  function createGround() {
    const groundGeometry = new THREE.BoxGeometry(1000, 1, 1000);
    const groundMaterial = new THREE.MeshStandardMaterial({ color: 0x808080 });
    const groundMesh = new THREE.Mesh(groundGeometry, groundMaterial);
    groundMesh.receiveShadow = true;
    groundMesh.position.y = -0.5;
    exteriorScene.add(groundMesh);
    
    const groundBodyDesc = RAPIER.RigidBodyDesc.fixed()
      .setTranslation(0, -0.5, 0);
    const groundBody = exteriorWorld.createRigidBody(groundBodyDesc);
    const groundColliderDesc = RAPIER.ColliderDesc.cuboid(500, 0.5, 500);
    exteriorWorld.createCollider(groundColliderDesc, groundBody);
  }
  
  let playerMesh;
  let movement = { forward: false, backward: false, left: false, right: false, jump: false };
  let shipMovement = { forward: false, backward: false, left: false, right: false, up: false, down: false };
  let stationMovement = { forward: false, backward: false, left: false, right: false, up: false, down: false };
  let isGrounded = $state(false);
  let jumpPressed = false;
  let canJump = true;
  
  // Debug HUD reactive states
  let playerPosition = $state({ x: 0, y: 0, z: 0 });
  let playerVelocity = $state({ x: 0, y: 0, z: 0 });
  let currentWorld = $derived(isInsideShip ? 'Ship Interior' : (isInsideStation ? 'Station Interior' : 'Exterior'));
  let shipWorld = $derived(isShipDocked ? 'Station Interior' : 'Exterior');
  let movementStatus = $derived(isGrounded ? 'Grounded' : 'Falling');
  let fps = $state(0);
  let frameCount = 0;
  let lastFpsTime = 0;
  
  function createPlayer() {
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    
    const capsuleGeometry = new THREE.CapsuleGeometry(capsuleRadius, capsuleHeight, 4, 8);
    const capsuleMaterial = new THREE.MeshStandardMaterial({ color: 0x00ff00 });
    playerMesh = new THREE.Mesh(capsuleGeometry, capsuleMaterial);
    playerMesh.castShadow = true;
    playerMesh.position.set(0, 3, 25);
    exteriorScene.add(playerMesh);
    
    const playerBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(0, 3, 25)
      .setCanSleep(false)
      .setLinearDamping(2.0)
      .setAngularDamping(5.0)
      .lockRotations();
    playerBody = exteriorWorld.createRigidBody(playerBodyDesc);
    
    const playerColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS)
      .setFriction(0.8)
      .setRestitution(0.1);
    playerCollider = exteriorWorld.createCollider(playerColliderDesc, playerBody);
    
    // Create interior player body - will be positioned when entering ship
    const interiorPlayerBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(0, -100, 0) // Start way below ground (inactive until entering ship)
      .setCanSleep(false)
      .setLinearDamping(2.0)
      .setAngularDamping(5.0)
      .lockRotations();
    interiorPlayerBody = interiorWorld.createRigidBody(interiorPlayerBodyDesc);
    
    const interiorPlayerColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS)
      .setFriction(0.8)
      .setRestitution(0.1);
    interiorPlayerCollider = interiorWorld.createCollider(interiorPlayerColliderDesc, interiorPlayerBody);
    
    // Create station player body - will be positioned when entering station
    const stationPlayerBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(0, -200, 0) // Start way below ground (inactive until entering station)
      .setCanSleep(false)
      .setLinearDamping(2.0)
      .setAngularDamping(5.0)
      .lockRotations();
    stationPlayerBody = stationWorld.createRigidBody(stationPlayerBodyDesc);
    
    const stationPlayerColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS)
      .setFriction(0.8)
      .setRestitution(0.1);
    stationPlayerCollider = stationWorld.createCollider(stationPlayerColliderDesc, stationPlayerBody);
  }
  
  let shipMesh;
  let shipInteriorMesh;
  let debugPlayerMesh;
  
  function createShip() {
    const shipGroup = new THREE.Group();
    const wallThickness = 0.4;
    const shipWidth = 20;
    const shipHeight = 6;
    const shipLength = 30;
    
    const hullMaterial = new THREE.MeshStandardMaterial({ color: 0x444444 });
    
    // Floor
    const floorGeometry = new THREE.BoxGeometry(shipWidth, wallThickness, shipLength);
    const floorMesh = new THREE.Mesh(floorGeometry, hullMaterial);
    floorMesh.castShadow = true;
    floorMesh.receiveShadow = true;
    floorMesh.position.set(0, wallThickness / 2, 0);
    shipGroup.add(floorMesh);
    
    // Ceiling
    const ceilingGeometry = new THREE.BoxGeometry(shipWidth, wallThickness, shipLength);
    const ceilingMesh = new THREE.Mesh(ceilingGeometry, hullMaterial);
    ceilingMesh.position.set(0, shipHeight - wallThickness / 2, 0);
    shipGroup.add(ceilingMesh);
    
    // Left wall
    const leftWallGeometry = new THREE.BoxGeometry(wallThickness, shipHeight, shipLength);
    const leftWallMesh = new THREE.Mesh(leftWallGeometry, hullMaterial);
    leftWallMesh.position.set(-shipWidth / 2 + wallThickness / 2, shipHeight / 2, 0);
    shipGroup.add(leftWallMesh);
    
    // Right wall
    const rightWallGeometry = new THREE.BoxGeometry(wallThickness, shipHeight, shipLength);
    const rightWallMesh = new THREE.Mesh(rightWallGeometry, hullMaterial);
    rightWallMesh.position.set(shipWidth / 2 - wallThickness / 2, shipHeight / 2, 0);
    shipGroup.add(rightWallMesh);
    
    // Back wall
    const backWallGeometry = new THREE.BoxGeometry(shipWidth, shipHeight, wallThickness);
    const backWallMesh = new THREE.Mesh(backWallGeometry, hullMaterial);
    backWallMesh.position.set(0, shipHeight / 2, -shipLength / 2 + wallThickness / 2);
    shipGroup.add(backWallMesh);
    
    // Front wall with door opening (bigger door)
    const doorWidth = 4;
    const doorHeight = 4;
    
    const frontWallLeft = new THREE.Mesh(
      new THREE.BoxGeometry((shipWidth - doorWidth) / 2, shipHeight, wallThickness),
      hullMaterial
    );
    frontWallLeft.position.set(-(shipWidth + doorWidth) / 4, shipHeight / 2, shipLength / 2 - wallThickness / 2);
    shipGroup.add(frontWallLeft);
    
    const frontWallRight = new THREE.Mesh(
      new THREE.BoxGeometry((shipWidth - doorWidth) / 2, shipHeight, wallThickness),
      hullMaterial
    );
    frontWallRight.position.set((shipWidth + doorWidth) / 4, shipHeight / 2, shipLength / 2 - wallThickness / 2);
    shipGroup.add(frontWallRight);
    
    const frontWallTop = new THREE.Mesh(
      new THREE.BoxGeometry(doorWidth, shipHeight - doorHeight, wallThickness),
      hullMaterial
    );
    frontWallTop.position.set(0, shipHeight - (shipHeight - doorHeight) / 2, shipLength / 2 - wallThickness / 2);
    shipGroup.add(frontWallTop);
    
    // Deck on top
    const deckGeometry = new THREE.BoxGeometry(shipWidth - 1, wallThickness, shipLength - 2);
    const deckMaterial = new THREE.MeshStandardMaterial({ color: 0x8B4513 });
    const deckMesh = new THREE.Mesh(deckGeometry, deckMaterial);
    deckMesh.position.y = shipHeight + wallThickness / 2;
    shipGroup.add(deckMesh);
    
    shipMesh = shipGroup;
    shipMesh.position.set(0, 0, 0);
    exteriorScene.add(shipMesh);
    
    const shipBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(0, 5, 0) // Start above ground
      .setCanSleep(false)
      .setLinearDamping(0.5)
      .setAngularDamping(2.0);
    shipBody = exteriorWorld.createRigidBody(shipBodyDesc);
    
    // Create compound colliders for each ship part in exterior world
    const floorCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, wallThickness / 2, shipLength / 2)
      .setTranslation(0, wallThickness / 2, 0);
    exteriorWorld.createCollider(floorCollider, shipBody);
    
    const ceilingCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, wallThickness / 2, shipLength / 2)
      .setTranslation(0, shipHeight - wallThickness / 2, 0);
    exteriorWorld.createCollider(ceilingCollider, shipBody);
    
    const leftWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, shipHeight / 2, shipLength / 2)
      .setTranslation(-shipWidth / 2 + wallThickness / 2, shipHeight / 2, 0);
    exteriorWorld.createCollider(leftWallCollider, shipBody);
    
    const rightWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, shipHeight / 2, shipLength / 2)
      .setTranslation(shipWidth / 2 - wallThickness / 2, shipHeight / 2, 0);
    exteriorWorld.createCollider(rightWallCollider, shipBody);
    
    const backWallCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, shipHeight / 2, wallThickness / 2)
      .setTranslation(0, shipHeight / 2, -shipLength / 2 + wallThickness / 2);
    exteriorWorld.createCollider(backWallCollider, shipBody);
    
    // Front wall colliders (with door opening)
    const frontWallLeftCollider = RAPIER.ColliderDesc.cuboid((shipWidth - doorWidth) / 4, shipHeight / 2, wallThickness / 2)
      .setTranslation(-(shipWidth + doorWidth) / 4, shipHeight / 2, shipLength / 2 - wallThickness / 2);
    exteriorWorld.createCollider(frontWallLeftCollider, shipBody);
    
    const frontWallRightCollider = RAPIER.ColliderDesc.cuboid((shipWidth - doorWidth) / 4, shipHeight / 2, wallThickness / 2)
      .setTranslation((shipWidth + doorWidth) / 4, shipHeight / 2, shipLength / 2 - wallThickness / 2);
    exteriorWorld.createCollider(frontWallRightCollider, shipBody);
    
    const frontWallTopCollider = RAPIER.ColliderDesc.cuboid(doorWidth / 2, (shipHeight - doorHeight) / 2, wallThickness / 2)
      .setTranslation(0, shipHeight - (shipHeight - doorHeight) / 2, shipLength / 2 - wallThickness / 2);
    exteriorWorld.createCollider(frontWallTopCollider, shipBody);

    // Create exterior door sensor attached to ship body for ship entry detection
    // Position it well inside the ship so player must be fully through the door
    const exteriorDoorSensorCollider = RAPIER.ColliderDesc.cuboid(doorWidth / 2 - 0.5, doorHeight / 2 - 0.5, 0.1)
      .setTranslation(0, doorHeight / 2, shipLength / 2 - wallThickness - 2.0)
      .setSensor(true)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS);
    exteriorWorld.createCollider(exteriorDoorSensorCollider, shipBody);
    
    // Create interior world physics (stationary ship interior)
    createInteriorPhysics(shipWidth, shipHeight, shipLength, wallThickness, doorWidth, doorHeight);
    
    createShipInterior();
  }
  
  function createInteriorPhysics(shipWidth, shipHeight, shipLength, wallThickness, doorWidth, doorHeight) {
    // Create stationary interior physics world at origin
    // Floor positioned at y = wallThickness/2 to match visual floor
    const interiorFloorBody = interiorWorld.createRigidBody(
      RAPIER.RigidBodyDesc.fixed().setTranslation(0, wallThickness / 2, 0)
    );
    const interiorFloorCollider = RAPIER.ColliderDesc.cuboid(
      shipWidth / 2 - wallThickness, 
      wallThickness / 2, 
      shipLength / 2 - wallThickness
    );
    interiorWorld.createCollider(interiorFloorCollider, interiorFloorBody);

    // Create sensor to detect when player enters door area in interior world
    const interiorDoorSensorBody = interiorWorld.createRigidBody(
      RAPIER.RigidBodyDesc.fixed().setTranslation(0, doorHeight / 2, shipLength / 2 - wallThickness)
    );
    const interiorDoorSensorCollider = RAPIER.ColliderDesc.cuboid(doorWidth / 2, doorHeight / 2, wallThickness / 2)
      .setSensor(true)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS);
    interiorWorld.createCollider(interiorDoorSensorCollider, interiorDoorSensorBody);
    
    console.log('Created interior floor at y:', wallThickness / 2, 'size:', {
      width: shipWidth - wallThickness * 2,
      height: wallThickness,
      length: shipLength - wallThickness * 2
    });
    
    // Interior walls
    const interiorLeftWallBody = interiorWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(-shipWidth / 2 + wallThickness / 2, shipHeight / 2, 0));
    const interiorLeftWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, shipHeight / 2, shipLength / 2);
    interiorWorld.createCollider(interiorLeftWallCollider, interiorLeftWallBody);
    
    const interiorRightWallBody = interiorWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(shipWidth / 2 - wallThickness / 2, shipHeight / 2, 0));
    const interiorRightWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, shipHeight / 2, shipLength / 2);
    interiorWorld.createCollider(interiorRightWallCollider, interiorRightWallBody);
    
    const interiorBackWallBody = interiorWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(0, shipHeight / 2, -shipLength / 2 + wallThickness / 2));
    const interiorBackWallCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, shipHeight / 2, wallThickness / 2);
    interiorWorld.createCollider(interiorBackWallCollider, interiorBackWallBody);
    
    // Front walls with door opening
    const interiorFrontLeftWallBody = interiorWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(-(shipWidth + doorWidth) / 4, shipHeight / 2, shipLength / 2 - wallThickness / 2));
    const interiorFrontLeftWallCollider = RAPIER.ColliderDesc.cuboid((shipWidth - doorWidth) / 4, shipHeight / 2, wallThickness / 2);
    interiorWorld.createCollider(interiorFrontLeftWallCollider, interiorFrontLeftWallBody);
    
    const interiorFrontRightWallBody = interiorWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation((shipWidth + doorWidth) / 4, shipHeight / 2, shipLength / 2 - wallThickness / 2));
    const interiorFrontRightWallCollider = RAPIER.ColliderDesc.cuboid((shipWidth - doorWidth) / 4, shipHeight / 2, wallThickness / 2);
    interiorWorld.createCollider(interiorFrontRightWallCollider, interiorFrontRightWallBody);
    
    const interiorFrontTopWallBody = interiorWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(0, shipHeight - (shipHeight - doorHeight) / 2, shipLength / 2 - wallThickness / 2));
    const interiorFrontTopWallCollider = RAPIER.ColliderDesc.cuboid(doorWidth / 2, (shipHeight - doorHeight) / 2, wallThickness / 2);
    interiorWorld.createCollider(interiorFrontTopWallCollider, interiorFrontTopWallBody);
  }
  
  function createShipInterior() {
    shipInteriorProxy = new THREE.Group();
    
    // Use actual ship dimensions for consistency
    const shipWidth = 20;
    const shipHeight = 6;
    const shipLength = 30;
    const wallThickness = 0.4;
    const doorWidth = 4;
    const doorHeight = 4;
    
    // Interior lighting
    const interiorLight = new THREE.PointLight(0xffffff, 0.8, 20);
    interiorLight.position.set(0, shipHeight / 2, 0);
    shipInteriorProxy.add(interiorLight);
    
    // No visual floor - physics floor is invisible
    
    // Interior walls (visual markers)
    const interiorWallMaterial = new THREE.MeshStandardMaterial({ 
      color: 0x666666,
      roughness: 0.6
    });
    
    // Clean interior without furniture for now
    
    // Add interior wall outlines to show actual boundaries
    const wallOutlineMaterial = new THREE.MeshStandardMaterial({ 
      color: 0x444444,
      transparent: true,
      opacity: 0.3
    });
    
    // Left wall outline
    const leftWallOutline = new THREE.Mesh(
      new THREE.BoxGeometry(wallThickness, shipHeight - wallThickness, shipLength - wallThickness),
      wallOutlineMaterial
    );
    leftWallOutline.position.set(-shipWidth / 2 + wallThickness / 2, shipHeight / 2, 0);
    shipInteriorProxy.add(leftWallOutline);
    
    // Right wall outline
    const rightWallOutline = new THREE.Mesh(
      new THREE.BoxGeometry(wallThickness, shipHeight - wallThickness, shipLength - wallThickness),
      wallOutlineMaterial
    );
    rightWallOutline.position.set(shipWidth / 2 - wallThickness / 2, shipHeight / 2, 0);
    shipInteriorProxy.add(rightWallOutline);
    
    // Back wall outline
    const backWallOutline = new THREE.Mesh(
      new THREE.BoxGeometry(shipWidth - wallThickness, shipHeight - wallThickness, wallThickness),
      wallOutlineMaterial
    );
    backWallOutline.position.set(0, shipHeight / 2, -shipLength / 2 + wallThickness / 2);
    shipInteriorProxy.add(backWallOutline);
    
    // Debug floor mesh removed - no visual floor needed, physics floor is invisible
    
    // Create debug player capsule representation for interior scene
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const debugCapsuleGeometry = new THREE.CapsuleGeometry(capsuleRadius, capsuleHeight, 4, 8);
    const debugCapsuleMaterial = new THREE.MeshStandardMaterial({ 
      color: 0x00ff00, 
      wireframe: true,
      transparent: true,
      opacity: 0.8
    });
    debugPlayerMesh = new THREE.Mesh(debugCapsuleGeometry, debugCapsuleMaterial);
    debugPlayerMesh.visible = false; // Hidden until player enters ship
    interiorScene.add(debugPlayerMesh);
    
    shipInteriorProxy.visible = false;
    interiorScene.add(shipInteriorProxy);
  }
  
  let stationMesh;
  let debugStationPlayerMesh;
  
  function createDockingStation() {
    const stationGroup = new THREE.Group();
    const wallThickness = 2.0;
    const stationWidth = 200;
    const stationHeight = 60;
    const stationLength = 250;
    const dockingBayWidth = 60;
    const dockingBayHeight = 30;
    const dockingBayDepth = 80;
    
    const stationMaterial = new THREE.MeshStandardMaterial({ color: 0x555555 });
    
    // Main station hull - hollow box
    // Floor
    const floorGeometry = new THREE.BoxGeometry(stationWidth, wallThickness, stationLength);
    const floorMesh = new THREE.Mesh(floorGeometry, stationMaterial);
    floorMesh.castShadow = true;
    floorMesh.receiveShadow = true;
    floorMesh.position.set(0, wallThickness / 2, 0);
    stationGroup.add(floorMesh);
    
    // Ceiling
    const ceilingMesh = new THREE.Mesh(floorGeometry, stationMaterial);
    ceilingMesh.position.set(0, stationHeight - wallThickness / 2, 0);
    stationGroup.add(ceilingMesh);
    
    // Walls (with docking bay opening)
    // Left wall
    const leftWallGeometry = new THREE.BoxGeometry(wallThickness, stationHeight, stationLength);
    const leftWallMesh = new THREE.Mesh(leftWallGeometry, stationMaterial);
    leftWallMesh.position.set(-stationWidth / 2 + wallThickness / 2, stationHeight / 2, 0);
    stationGroup.add(leftWallMesh);
    
    // Right wall
    const rightWallMesh = new THREE.Mesh(leftWallGeometry, stationMaterial);
    rightWallMesh.position.set(stationWidth / 2 - wallThickness / 2, stationHeight / 2, 0);
    stationGroup.add(rightWallMesh);
    
    // Back wall
    const backWallGeometry = new THREE.BoxGeometry(stationWidth, stationHeight, wallThickness);
    const backWallMesh = new THREE.Mesh(backWallGeometry, stationMaterial);
    backWallMesh.position.set(0, stationHeight / 2, -stationLength / 2 + wallThickness / 2);
    stationGroup.add(backWallMesh);
    
    // Front wall with docking bay opening
    const frontWallLeft = new THREE.Mesh(
      new THREE.BoxGeometry((stationWidth - dockingBayWidth) / 2, stationHeight, wallThickness),
      stationMaterial
    );
    frontWallLeft.position.set(-(stationWidth + dockingBayWidth) / 4, stationHeight / 2, stationLength / 2 - wallThickness / 2);
    stationGroup.add(frontWallLeft);
    
    const frontWallRight = new THREE.Mesh(
      new THREE.BoxGeometry((stationWidth - dockingBayWidth) / 2, stationHeight, wallThickness),
      stationMaterial
    );
    frontWallRight.position.set((stationWidth + dockingBayWidth) / 4, stationHeight / 2, stationLength / 2 - wallThickness / 2);
    stationGroup.add(frontWallRight);
    
    const frontWallTop = new THREE.Mesh(
      new THREE.BoxGeometry(dockingBayWidth, stationHeight - dockingBayHeight, wallThickness),
      stationMaterial
    );
    frontWallTop.position.set(0, stationHeight - (stationHeight - dockingBayHeight) / 2, stationLength / 2 - wallThickness / 2);
    stationGroup.add(frontWallTop);
    
    stationMesh = stationGroup;
    // Position station on opposite side from ship's door (ship door faces +Z, so put station at -Z)
    stationMesh.position.set(0, 40, -400); // Behind the ship, higher up
    exteriorScene.add(stationMesh);
    
    // Create station physics body as dynamic for movement with gravity
    const stationBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(0, 40, -400)
      .setCanSleep(false)
      .setLinearDamping(0.8)
      .setAngularDamping(2.0);
    stationBody = exteriorWorld.createRigidBody(stationBodyDesc);
    
    // Station colliders - full structure for proper physics
    // Floor
    const stationFloorCollider = RAPIER.ColliderDesc.cuboid(stationWidth / 2, wallThickness / 2, stationLength / 2)
      .setTranslation(0, wallThickness / 2, 0);
    exteriorWorld.createCollider(stationFloorCollider, stationBody);
    
    // Ceiling  
    const stationCeilingCollider = RAPIER.ColliderDesc.cuboid(stationWidth / 2, wallThickness / 2, stationLength / 2)
      .setTranslation(0, stationHeight - wallThickness / 2, 0);
    exteriorWorld.createCollider(stationCeilingCollider, stationBody);
    
    // Left wall
    const stationLeftWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, stationHeight / 2, stationLength / 2)
      .setTranslation(-stationWidth / 2 + wallThickness / 2, stationHeight / 2, 0);
    exteriorWorld.createCollider(stationLeftWallCollider, stationBody);
    
    // Right wall
    const stationRightWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, stationHeight / 2, stationLength / 2)
      .setTranslation(stationWidth / 2 - wallThickness / 2, stationHeight / 2, 0);
    exteriorWorld.createCollider(stationRightWallCollider, stationBody);
    
    // Back wall
    const stationBackWallCollider = RAPIER.ColliderDesc.cuboid(stationWidth / 2, stationHeight / 2, wallThickness / 2)
      .setTranslation(0, stationHeight / 2, -stationLength / 2 + wallThickness / 2);
    exteriorWorld.createCollider(stationBackWallCollider, stationBody);
    
    // Front walls (with docking bay opening)
    const stationFrontLeftCollider = RAPIER.ColliderDesc.cuboid((stationWidth - dockingBayWidth) / 4, stationHeight / 2, wallThickness / 2)
      .setTranslation(-(stationWidth + dockingBayWidth) / 4, stationHeight / 2, stationLength / 2 - wallThickness / 2);
    exteriorWorld.createCollider(stationFrontLeftCollider, stationBody);
    
    const stationFrontRightCollider = RAPIER.ColliderDesc.cuboid((stationWidth - dockingBayWidth) / 4, stationHeight / 2, wallThickness / 2)
      .setTranslation((stationWidth + dockingBayWidth) / 4, stationHeight / 2, stationLength / 2 - wallThickness / 2);
    exteriorWorld.createCollider(stationFrontRightCollider, stationBody);
    
    const stationFrontTopCollider = RAPIER.ColliderDesc.cuboid(dockingBayWidth / 2, (stationHeight - dockingBayHeight) / 2, wallThickness / 2)
      .setTranslation(0, stationHeight - (stationHeight - dockingBayHeight) / 2, stationLength / 2 - wallThickness / 2);
    exteriorWorld.createCollider(stationFrontTopCollider, stationBody);
    
    // Create ship docking sensor just inside the docking bay entrance
    const dockingSensorCollider = RAPIER.ColliderDesc.cuboid(dockingBayWidth / 2 - 1, dockingBayHeight / 2 - 1, 5)
      .setTranslation(0, dockingBayHeight / 2, stationLength / 2 - 10)
      .setSensor(true)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS);
    exteriorWorld.createCollider(dockingSensorCollider, stationBody);
    
    // Add debug visualization for docking sensor
    const dockingSensorGeometry = new THREE.BoxGeometry(
      (dockingBayWidth / 2 - 1) * 2,
      (dockingBayHeight / 2 - 1) * 2,
      10
    );
    const dockingSensorMaterial = new THREE.MeshBasicMaterial({
      color: 0x00ff00,
      transparent: true,
      opacity: 0.3,
      wireframe: true
    });
    const dockingSensorMesh = new THREE.Mesh(dockingSensorGeometry, dockingSensorMaterial);
    dockingSensorMesh.position.set(0, dockingBayHeight / 2, stationLength / 2 - 10);
    stationGroup.add(dockingSensorMesh);
    
    // Create player entry sensor for direct station access
    const stationEntrySensorCollider = RAPIER.ColliderDesc.cuboid(3, 3, 0.5)
      .setTranslation(-10, 3, stationLength / 2 - wallThickness / 2)
      .setSensor(true)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS);
    exteriorWorld.createCollider(stationEntrySensorCollider, stationBody);
    
    createStationInterior();
  }
  
  let stationShipProxy; // Ship representation in station interior

  function createStationInterior() {
    stationInteriorProxy = new THREE.Group();
    
    const stationWidth = 200;
    const stationHeight = 60;
    const stationLength = 250;
    const wallThickness = 2.0;
    
    // Station interior lighting
    const stationInteriorLight = new THREE.PointLight(0xffffff, 1.2, 150);
    stationInteriorLight.position.set(0, stationHeight / 2, 0);
    stationInteriorProxy.add(stationInteriorLight);
    
    // Add additional lights for better visibility
    const frontLight = new THREE.PointLight(0xffffff, 0.8, 100);
    frontLight.position.set(0, stationHeight / 2, stationLength / 3);
    stationInteriorProxy.add(frontLight);
    
    const backLight = new THREE.PointLight(0xffffff, 0.8, 100);
    backLight.position.set(0, stationHeight / 2, -stationLength / 3);
    stationInteriorProxy.add(backLight);
    
    // Create visual walls for station interior
    const interiorWallMaterial = new THREE.MeshStandardMaterial({ color: 0x666666 });
    
    // Floor
    const floorGeometry = new THREE.BoxGeometry(stationWidth - wallThickness, wallThickness, stationLength - wallThickness);
    const floorMesh = new THREE.Mesh(floorGeometry, interiorWallMaterial);
    floorMesh.position.y = wallThickness / 2;
    stationInteriorProxy.add(floorMesh);
    
    // Ceiling (partial to show it's a hangar)
    const ceilingGeometry = new THREE.BoxGeometry(stationWidth - wallThickness, wallThickness, stationLength - wallThickness);
    const ceilingMesh = new THREE.Mesh(ceilingGeometry, interiorWallMaterial);
    ceilingMesh.position.y = stationHeight - wallThickness / 2;
    stationInteriorProxy.add(ceilingMesh);
    
    // Left wall
    const leftWallGeometry = new THREE.BoxGeometry(wallThickness, stationHeight, stationLength);
    const leftWallMesh = new THREE.Mesh(leftWallGeometry, interiorWallMaterial);
    leftWallMesh.position.set(-stationWidth / 2 + wallThickness / 2, stationHeight / 2, 0);
    stationInteriorProxy.add(leftWallMesh);
    
    // Right wall
    const rightWallGeometry = new THREE.BoxGeometry(wallThickness, stationHeight, stationLength);
    const rightWallMesh = new THREE.Mesh(rightWallGeometry, interiorWallMaterial);
    rightWallMesh.position.set(stationWidth / 2 - wallThickness / 2, stationHeight / 2, 0);
    stationInteriorProxy.add(rightWallMesh);
    
    // Back wall
    const backWallGeometry = new THREE.BoxGeometry(stationWidth, stationHeight, wallThickness);
    const backWallMesh = new THREE.Mesh(backWallGeometry, interiorWallMaterial);
    backWallMesh.position.set(0, stationHeight / 2, -stationLength / 2 + wallThickness / 2);
    stationInteriorProxy.add(backWallMesh);
    
    // Front wall with docking bay opening (partial walls around opening)
    const dockingBayWidth = 60;
    const dockingBayHeight = 30;
    
    // Front wall sections around docking bay
    const frontWallLeft = new THREE.Mesh(
      new THREE.BoxGeometry((stationWidth - dockingBayWidth) / 2, stationHeight, wallThickness),
      interiorWallMaterial
    );
    frontWallLeft.position.set(-(stationWidth + dockingBayWidth) / 4, stationHeight / 2, stationLength / 2 - wallThickness / 2);
    stationInteriorProxy.add(frontWallLeft);
    
    const frontWallRight = new THREE.Mesh(
      new THREE.BoxGeometry((stationWidth - dockingBayWidth) / 2, stationHeight, wallThickness),
      interiorWallMaterial
    );
    frontWallRight.position.set((stationWidth + dockingBayWidth) / 4, stationHeight / 2, stationLength / 2 - wallThickness / 2);
    stationInteriorProxy.add(frontWallRight);
    
    const frontWallTop = new THREE.Mesh(
      new THREE.BoxGeometry(dockingBayWidth, stationHeight - dockingBayHeight, wallThickness),
      interiorWallMaterial
    );
    frontWallTop.position.set(0, stationHeight - (stationHeight - dockingBayHeight) / 2, stationLength / 2 - wallThickness / 2);
    stationInteriorProxy.add(frontWallTop);
    
    // Add some interior structures (platforms, etc.)
    const platformMaterial = new THREE.MeshStandardMaterial({ color: 0x777777 });
    const platform = new THREE.Mesh(
      new THREE.BoxGeometry(60, 4, 80),
      platformMaterial
    );
    platform.position.set(-40, 4, -50);
    stationInteriorProxy.add(platform);
    
    // Create ship proxy for station interior view
    createStationShipProxy();
    
    // Create debug player mesh for station
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const debugCapsuleGeometry = new THREE.CapsuleGeometry(capsuleRadius, capsuleHeight, 4, 8);
    const debugCapsuleMaterial = new THREE.MeshStandardMaterial({ 
      color: 0xff0000, 
      wireframe: true,
      transparent: true,
      opacity: 0.8
    });
    debugStationPlayerMesh = new THREE.Mesh(debugCapsuleGeometry, debugCapsuleMaterial);
    debugStationPlayerMesh.visible = false;
    stationScene.add(debugStationPlayerMesh);
    
    // Create station interior physics
    createStationPhysics(stationWidth, stationHeight, stationLength, wallThickness);
    
    stationInteriorProxy.visible = false;
    stationScene.add(stationInteriorProxy);
  }
  
  function createStationPhysics(stationWidth, stationHeight, stationLength, wallThickness) {
    // Station interior floor
    const stationFloorBody = stationWorld.createRigidBody(
      RAPIER.RigidBodyDesc.fixed().setTranslation(0, wallThickness / 2, 0)
    );
    const stationFloorCollider = RAPIER.ColliderDesc.cuboid(
      stationWidth / 2 - wallThickness, 
      wallThickness / 2, 
      stationLength / 2 - wallThickness
    );
    stationWorld.createCollider(stationFloorCollider, stationFloorBody);
    
    // Station interior walls
    const stationLeftWallBody = stationWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(-stationWidth / 2 + wallThickness / 2, stationHeight / 2, 0));
    const stationLeftWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, stationHeight / 2, stationLength / 2);
    stationWorld.createCollider(stationLeftWallCollider, stationLeftWallBody);
    
    const stationRightWallBody = stationWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(stationWidth / 2 - wallThickness / 2, stationHeight / 2, 0));
    const stationRightWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, stationHeight / 2, stationLength / 2);
    stationWorld.createCollider(stationRightWallCollider, stationRightWallBody);
    
    // Station interior back wall
    const stationBackWallBody = stationWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(0, stationHeight / 2, -stationLength / 2 + wallThickness / 2));
    const stationBackWallCollider = RAPIER.ColliderDesc.cuboid(stationWidth / 2, stationHeight / 2, wallThickness / 2);
    stationWorld.createCollider(stationBackWallCollider, stationBackWallBody);
    
    // Station interior front walls (with docking bay opening)
    const dockingBayWidth = 60, dockingBayHeight = 30;
    
    // Front left wall section
    const stationFrontLeftWallBody = stationWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(-(stationWidth + dockingBayWidth) / 4, stationHeight / 2, stationLength / 2 - wallThickness / 2));
    const stationFrontLeftWallCollider = RAPIER.ColliderDesc.cuboid((stationWidth - dockingBayWidth) / 4, stationHeight / 2, wallThickness / 2);
    stationWorld.createCollider(stationFrontLeftWallCollider, stationFrontLeftWallBody);
    
    // Front right wall section
    const stationFrontRightWallBody = stationWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation((stationWidth + dockingBayWidth) / 4, stationHeight / 2, stationLength / 2 - wallThickness / 2));
    const stationFrontRightWallCollider = RAPIER.ColliderDesc.cuboid((stationWidth - dockingBayWidth) / 4, stationHeight / 2, wallThickness / 2);
    stationWorld.createCollider(stationFrontRightWallCollider, stationFrontRightWallBody);
    
    // Front top wall section (above docking bay opening)
    const stationFrontTopWallBody = stationWorld.createRigidBody(RAPIER.RigidBodyDesc.fixed().setTranslation(0, stationHeight - (stationHeight - dockingBayHeight) / 2, stationLength / 2 - wallThickness / 2));
    const stationFrontTopWallCollider = RAPIER.ColliderDesc.cuboid(dockingBayWidth / 2, (stationHeight - dockingBayHeight) / 2, wallThickness / 2);
    stationWorld.createCollider(stationFrontTopWallCollider, stationFrontTopWallBody);
    
    // Station interior ceiling
    const stationCeilingBody = stationWorld.createRigidBody(
      RAPIER.RigidBodyDesc.fixed().setTranslation(0, stationHeight - wallThickness / 2, 0)
    );
    const stationCeilingCollider = RAPIER.ColliderDesc.cuboid(
      stationWidth / 2 - wallThickness, 
      wallThickness / 2, 
      stationLength / 2 - wallThickness
    );
    stationWorld.createCollider(stationCeilingCollider, stationCeilingBody);
    
    console.log('Created station interior physics');
    
    // Create ship physics body in station world (initially inactive)
    const stationShipBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(0, -300, 0) // Start way below (inactive until ship docks)
      .setCanSleep(false)
      .setLinearDamping(0.5)
      .setAngularDamping(2.0);
    stationShipBody = stationWorld.createRigidBody(stationShipBodyDesc);
    
    // Create compound ship colliders in station world (same as exterior with door opening)
    const shipWidth = 20, shipHeight = 6, shipLength = 30;
    const doorWidth = 4, doorHeight = 4;
    
    // Ship floor in station (positioned to align with station floor for seamless walking)
    const shipFloorInStationCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, wallThickness / 2, shipLength / 2)
      .setTranslation(0, -wallThickness, 0); // Position ship floor 1 unit below ship body to align with station floor
    stationWorld.createCollider(shipFloorInStationCollider, stationShipBody);
    
    // Ship ceiling in station  
    const shipCeilingInStationCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, wallThickness / 2, shipLength / 2)
      .setTranslation(0, shipHeight - wallThickness, 0); // Adjusted for floor offset
    stationWorld.createCollider(shipCeilingInStationCollider, stationShipBody);
    
    // Ship left wall in station
    const shipLeftWallInStationCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, shipHeight / 2, shipLength / 2)
      .setTranslation(-shipWidth / 2 + wallThickness / 2, shipHeight / 2 - wallThickness, 0); // Adjusted for floor offset
    stationWorld.createCollider(shipLeftWallInStationCollider, stationShipBody);
    
    // Ship right wall in station  
    const shipRightWallInStationCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, shipHeight / 2, shipLength / 2)
      .setTranslation(shipWidth / 2 - wallThickness / 2, shipHeight / 2 - wallThickness, 0); // Adjusted for floor offset
    stationWorld.createCollider(shipRightWallInStationCollider, stationShipBody);
    
    // Ship back wall in station
    const shipBackWallInStationCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, shipHeight / 2, wallThickness / 2)
      .setTranslation(0, shipHeight / 2 - wallThickness, -shipLength / 2 + wallThickness / 2); // Adjusted for floor offset
    stationWorld.createCollider(shipBackWallInStationCollider, stationShipBody);
    
    // Ship front wall colliders in station (with door opening)
    const shipFrontWallLeftInStationCollider = RAPIER.ColliderDesc.cuboid((shipWidth - doorWidth) / 4, shipHeight / 2, wallThickness / 2)
      .setTranslation(-(shipWidth + doorWidth) / 4, shipHeight / 2 - wallThickness, shipLength / 2 - wallThickness / 2); // Adjusted for floor offset
    stationWorld.createCollider(shipFrontWallLeftInStationCollider, stationShipBody);
    
    const shipFrontWallRightInStationCollider = RAPIER.ColliderDesc.cuboid((shipWidth - doorWidth) / 4, shipHeight / 2, wallThickness / 2)
      .setTranslation((shipWidth + doorWidth) / 4, shipHeight / 2 - wallThickness, shipLength / 2 - wallThickness / 2); // Adjusted for floor offset
    stationWorld.createCollider(shipFrontWallRightInStationCollider, stationShipBody);
    
    const shipFrontWallTopInStationCollider = RAPIER.ColliderDesc.cuboid(doorWidth / 2, (shipHeight - doorHeight) / 2, wallThickness / 2)
      .setTranslation(0, shipHeight - (shipHeight - doorHeight) / 2 - wallThickness, shipLength / 2 - wallThickness / 2); // Adjusted for floor offset
    stationWorld.createCollider(shipFrontWallTopInStationCollider, stationShipBody);
    
    // Note: Door sensor removed - using position-based detection instead
  }
  
  function createStationShipProxy() {
    // Create a copy of the ship for display in the station interior
    const shipGroup = new THREE.Group();
    const wallThickness = 0.4;
    const shipWidth = 20;
    const shipHeight = 6;
    const shipLength = 30;
    
    const hullMaterial = new THREE.MeshStandardMaterial({ color: 0x444444 });
    
    // Floor (matched exactly to physics collider position)
    const floorGeometry = new THREE.BoxGeometry(shipWidth, wallThickness, shipLength);
    const floorMesh = new THREE.Mesh(floorGeometry, hullMaterial);
    floorMesh.castShadow = true;
    floorMesh.receiveShadow = true;
    // Physics collider is at Y=-wallThickness, visual mesh center matches
    floorMesh.position.set(0, -wallThickness, 0); 
    shipGroup.add(floorMesh);
    
    // Ceiling (matched exactly to physics collider position)  
    const ceilingGeometry = new THREE.BoxGeometry(shipWidth, wallThickness, shipLength);
    const ceilingMesh = new THREE.Mesh(ceilingGeometry, hullMaterial);
    // Physics collider is at Y=shipHeight-wallThickness, visual mesh center matches
    ceilingMesh.position.set(0, shipHeight - wallThickness, 0);
    shipGroup.add(ceilingMesh);
    
    // Left wall (matched exactly to physics collider position)
    const leftWallGeometry = new THREE.BoxGeometry(wallThickness, shipHeight, shipLength);
    const leftWallMesh = new THREE.Mesh(leftWallGeometry, hullMaterial);
    // Physics collider is at Y=shipHeight/2-wallThickness, visual mesh center matches
    leftWallMesh.position.set(-shipWidth / 2 + wallThickness / 2, shipHeight / 2 - wallThickness, 0);
    shipGroup.add(leftWallMesh);
    
    // Right wall (matched exactly to physics collider position)
    const rightWallMesh = new THREE.Mesh(leftWallGeometry, hullMaterial);
    rightWallMesh.position.set(shipWidth / 2 - wallThickness / 2, shipHeight / 2 - wallThickness, 0);
    shipGroup.add(rightWallMesh);
    
    // Back wall (matched exactly to physics collider position)
    const backWallGeometry = new THREE.BoxGeometry(shipWidth, shipHeight, wallThickness);
    const backWallMesh = new THREE.Mesh(backWallGeometry, hullMaterial);
    backWallMesh.position.set(0, shipHeight / 2 - wallThickness, -shipLength / 2 + wallThickness / 2);
    shipGroup.add(backWallMesh);
    
    // Front wall with door opening (adjusted for floor at Y=0)
    const doorWidth = 4;  
    const doorHeight = 4;
    
    const frontWallLeft = new THREE.Mesh(
      new THREE.BoxGeometry((shipWidth - doorWidth) / 2, shipHeight, wallThickness),
      hullMaterial
    );
    frontWallLeft.position.set(-(shipWidth + doorWidth) / 4, shipHeight / 2 - wallThickness, shipLength / 2 - wallThickness / 2);
    shipGroup.add(frontWallLeft);
    
    const frontWallRight = new THREE.Mesh(
      new THREE.BoxGeometry((shipWidth - doorWidth) / 2, shipHeight, wallThickness),
      hullMaterial
    );
    frontWallRight.position.set((shipWidth + doorWidth) / 4, shipHeight / 2 - wallThickness, shipLength / 2 - wallThickness / 2);
    shipGroup.add(frontWallRight);
    
    const frontWallTop = new THREE.Mesh(
      new THREE.BoxGeometry(doorWidth, shipHeight - doorHeight, wallThickness),
      hullMaterial
    );
    // Matched exactly to physics collider position  
    frontWallTop.position.set(0, shipHeight - (shipHeight - doorHeight) / 2 - wallThickness, shipLength / 2 - wallThickness / 2);
    shipGroup.add(frontWallTop);
    
    // Deck on top
    const deckGeometry = new THREE.BoxGeometry(shipWidth - 1, wallThickness, shipLength - 2);
    const deckMaterial = new THREE.MeshStandardMaterial({ color: 0x8B4513 });
    const deckMesh = new THREE.Mesh(deckGeometry, deckMaterial);
    deckMesh.position.y = shipHeight + wallThickness / 2;
    shipGroup.add(deckMesh);
    
    stationShipProxy = shipGroup;
    stationShipProxy.visible = false; // Hidden until ship docks
    stationScene.add(stationShipProxy);
  }
  
  function handleKeyDown(event) {
    switch(event.key.toLowerCase()) {
      case 'w': movement.forward = true; break;
      case 's': movement.backward = true; break;
      case 'a': movement.left = true; break;
      case 'd': movement.right = true; break;
      case ' ': 
        if (!jumpPressed && canJump) {
          movement.jump = true;
          jumpPressed = true;
        }
        break;
      case 'arrowup': 
        shipMovement.forward = true; 
        console.log('Arrow UP pressed - forward thrust');
        break;
      case 'arrowdown': 
        shipMovement.backward = true; 
        console.log('Arrow DOWN pressed - backward thrust');
        break;
      case 'arrowleft': 
        shipMovement.left = true; 
        console.log('Arrow LEFT pressed - left yaw');
        break;
      case 'arrowright': 
        shipMovement.right = true; 
        console.log('Arrow RIGHT pressed - right yaw');
        break;
      case 'q': 
        shipMovement.up = true; 
        console.log('Q pressed');
        break;
      case 'e': 
        shipMovement.down = true; 
        console.log('E pressed');
        break;
      // Ship movement controls (TFGH) - only when player is inside ship
      case 't':
        if (isInsideShip) {
          shipMovement.forward = true;
          console.log('T pressed - ship forward');
        }
        break;
      case 'g':
        if (isInsideShip) {
          shipMovement.backward = true;
          console.log('G pressed - ship backward');
        }
        break;
      case 'f':
        if (isInsideShip) {
          shipMovement.left = true;
          console.log('F pressed - ship left');
        }
        break;
      case 'h':
        if (isInsideShip) {
          shipMovement.right = true;
          console.log('H pressed - ship right');
        }
        break;
      case 'r':
        if (isInsideShip) {
          shipMovement.down = true;
          console.log('R pressed - ship down');
        }
        break;
      case 'y':
        if (isInsideShip) {
          shipMovement.up = true;
          console.log('Y pressed - ship up');
        }
        break;
      // Station movement controls (IKJL) - only when player is inside station
      case 'i':
        if (isInsideStation) {
          stationMovement.forward = true;
          console.log('I pressed - station forward');
        }
        break;
      case 'k':
        if (isInsideStation) {
          stationMovement.backward = true;
          console.log('K pressed - station backward');
        }
        break;
      case 'j':
        if (isInsideStation) {
          stationMovement.left = true;
          console.log('J pressed - station left');
        }
        break;
      case 'l':
        if (isInsideStation) {
          stationMovement.right = true;
          console.log('L pressed - station right');
        }
        break;
      case 'o':
        isFirstPerson = !isFirstPerson;
        console.log('Camera switched to:', isFirstPerson ? 'First Person' : 'Third Person');
        
        // Reset mouse look rotation when switching to first person
        if (isFirstPerson && !isPointerLocked) {
          cameraRotation.yaw = 0;
          cameraRotation.pitch = 0;
        }
        
        // Exit pointer lock when switching to third person
        if (!isFirstPerson && isPointerLocked) {
          document.exitPointerLock();
        }
        break;
      case 'escape':
        if (isPointerLocked) {
          document.exitPointerLock();
        }
        break;
      case 'x':
        // Exit station (temporary key for testing)
        if (isInsideStation) {
          console.log('X pressed - exiting station');
          exitStation();
        }
        break;
    }
  }
  
  function handleClick() {
    if (!isPointerLocked && isFirstPerson) {
      // Request pointer lock when clicking in first person mode
      canvas.requestPointerLock();
    }
  }

  function handlePointerLockChange() {
    isPointerLocked = document.pointerLockElement === canvas;
    console.log('Pointer lock:', isPointerLocked ? 'ON' : 'OFF');
    
    // Reset camera rotation when exiting pointer lock to prevent glitches
    if (!isPointerLocked) {
      cameraRotation.yaw = 0;
      cameraRotation.pitch = 0;
    }
  }

  function handleMouseMove(event) {
    if (!isPointerLocked || !isFirstPerson) return;
    
    const sensitivity = 0.002;
    const deltaX = event.movementX * sensitivity;
    const deltaY = event.movementY * sensitivity;
    
    // Update camera rotation
    cameraRotation.yaw += deltaX;
    cameraRotation.pitch -= deltaY;
    
    // Clamp vertical rotation to prevent over-rotation
    cameraRotation.pitch = Math.max(-Math.PI / 2, Math.min(Math.PI / 2, cameraRotation.pitch));
    
    // Keep yaw in reasonable bounds (optional, helps prevent numerical drift)
    while (cameraRotation.yaw > Math.PI) cameraRotation.yaw -= 2 * Math.PI;
    while (cameraRotation.yaw < -Math.PI) cameraRotation.yaw += 2 * Math.PI;
  }

  function handleKeyUp(event) {
    switch(event.key.toLowerCase()) {
      case 'w': movement.forward = false; break;
      case 's': movement.backward = false; break;
      case 'a': movement.left = false; break;
      case 'd': movement.right = false; break;
      case ' ': 
        movement.jump = false;
        jumpPressed = false;
        break;
      case 'arrowup': shipMovement.forward = false; break;
      case 'arrowdown': shipMovement.backward = false; break;
      case 'arrowleft': shipMovement.left = false; break;
      case 'arrowright': shipMovement.right = false; break;
      case 'q': shipMovement.up = false; break;
      case 'e': shipMovement.down = false; break;
      // Ship movement controls (TFGH + RY)
      case 't': shipMovement.forward = false; break;
      case 'g': shipMovement.backward = false; break;
      case 'f': shipMovement.left = false; break;
      case 'h': shipMovement.right = false; break;
      case 'r': shipMovement.down = false; break;
      case 'y': shipMovement.up = false; break;
      // Station movement controls (IKJL)
      case 'i': stationMovement.forward = false; break;
      case 'k': stationMovement.backward = false; break;
      case 'j': stationMovement.left = false; break;
      case 'l': stationMovement.right = false; break;
    }
  }
  
  
  function immediateEnterInterior() {
    if (isEntering || isInsideShip) return; // Prevent multiple calls and double entry
    isEntering = true;
    console.log('Instant entry to interior - converting dynamic to kinematic');
    
    // Additional safety check - ensure we have valid bodies
    if (!playerBody || !playerCollider || !interiorPlayerBody) {
      console.error('Invalid player bodies during entry');
      isEntering = false;
      return;
    }
    
    // Get current exterior state
    const currentExteriorPos = playerBody.translation();
    const currentExteriorVel = playerBody.linvel();
    const shipPos = shipBody.translation();
    const shipRot = shipBody.rotation();
    const shipQuat = new THREE.Quaternion(shipRot.x, shipRot.y, shipRot.z, shipRot.w);
    
    // Calculate relative position inside ship (exterior pos - ship pos, rotated by inverse ship rotation)
    const worldRelativePos = new THREE.Vector3(
      currentExteriorPos.x - shipPos.x,
      currentExteriorPos.y - shipPos.y, 
      currentExteriorPos.z - shipPos.z
    );
    
    // Apply inverse ship rotation to get interior relative position
    const shipQuatInverse = shipQuat.clone().invert();
    worldRelativePos.applyQuaternion(shipQuatInverse);
    
    console.log('Interior relative position:', worldRelativePos);
    
    // Natural entry - let sensor collision determine when to switch to interior physics
    console.log('Proceeding with natural interior entry at position:', worldRelativePos);
    
    // Use natural Y position from the transformation
    
    // Create new kinematic body FIRST at current exterior position
    const kinematicBodyDesc = RAPIER.RigidBodyDesc.kinematicPositionBased()
      .setTranslation(currentExteriorPos.x, currentExteriorPos.y, currentExteriorPos.z);
    const newKinematicBody = exteriorWorld.createRigidBody(kinematicBodyDesc);
    
    // Create new collider for kinematic body - make it a sensor to avoid physics interactions
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const newColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setSensor(true); // Sensor mode - no collision response, just detection
    const newCollider = exteriorWorld.createCollider(newColliderDesc, newKinematicBody);
    
    console.log('New kinematic body and collider created');
    
    // Remove old dynamic body and collider safely
    if (playerCollider) {
      exteriorWorld.removeCollider(playerCollider, true); // Wake up neighbors
    }
    if (playerBody) {
      exteriorWorld.removeRigidBody(playerBody);
    }
    
    // Switch to new kinematic body
    playerBody = newKinematicBody;
    playerCollider = newCollider;
    isInsideShip = true;
    
    // Set interior player to the natural relative position
    interiorPlayerBody.setTranslation({ 
      x: worldRelativePos.x, 
      y: worldRelativePos.y, 
      z: worldRelativePos.z 
    }, true);
    
    // Transform velocity to interior coordinate system
    const interiorVel = new THREE.Vector3(currentExteriorVel.x, currentExteriorVel.y, currentExteriorVel.z);
    interiorVel.applyQuaternion(shipQuatInverse);
    interiorPlayerBody.setLinvel({ x: interiorVel.x, y: interiorVel.y, z: interiorVel.z }, true);
    
    console.log('Instantly entered interior - kinematic body type:', playerBody.bodyType());
    console.log('Interior player position:', interiorPlayerBody.translation());
    console.log('Interior player velocity:', interiorPlayerBody.linvel());
    
    // Reset entering flag
    isEntering = false;
  }

  function enterShip() {
    console.log('Seamlessly entering ship - switching to interior physics control');
    isInsideShip = true;
    
    // Get current exterior player state
    const currentExteriorPos = playerBody.translation();
    const currentExteriorVel = playerBody.linvel();
    const shipPos = shipBody.translation();
    
    // Calculate relative position inside ship (exterior pos - ship pos)
    const relativeX = currentExteriorPos.x - shipPos.x;
    const relativeY = currentExteriorPos.y - shipPos.y;
    const relativeZ = currentExteriorPos.z - shipPos.z;
    
    console.log('Switching control - relative position:', { relativeX, relativeY, relativeZ });
    
    // Convert exterior player to kinematic (synchronized by interior simulation)
    if (playerCollider) {
      exteriorWorld.removeCollider(playerCollider, true);
    }
    if (playerBody) {
      exteriorWorld.removeRigidBody(playerBody);
    }
    
    const kinematicPlayerBodyDesc = RAPIER.RigidBodyDesc.kinematicPositionBased()
      .setTranslation(currentExteriorPos.x, currentExteriorPos.y, currentExteriorPos.z);
    playerBody = exteriorWorld.createRigidBody(kinematicPlayerBodyDesc);
    
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const kinematicPlayerColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setSensor(true); // Sensor mode - no collision response with ship
    playerCollider = exteriorWorld.createCollider(kinematicPlayerColliderDesc, playerBody);
    
    // Set interior player to relative position and maintain velocity
    interiorPlayerBody.setTranslation({ x: relativeX, y: relativeY, z: relativeZ }, true);
    interiorPlayerBody.setLinvel({ x: currentExteriorVel.x, y: currentExteriorVel.y, z: currentExteriorVel.z }, true);
    
    console.log('Interior player taking control at:', interiorPlayerBody.translation());
    console.log('Expected relative position should be:', { relativeX, relativeY, relativeZ });
    console.log('Ship position was:', shipPos);
    console.log('Exterior position was:', currentExteriorPos);
    
    // Verify the position was set correctly
    setTimeout(() => {
      console.log('Interior player position after frame:', interiorPlayerBody.translation());
    }, 100);
  }
  
  function startEnterTransition() {
    if (isTransitioning) return;
    console.log('Starting ship entry transition');
    
    isTransitioning = true;
    transitionProgress = 0;
    
    // Immediately position interior player body at correct location to avoid glitch
    const currentExteriorPos = playerBody.translation();
    const shipPos = shipBody.translation();
    
    const relativeX = currentExteriorPos.x - shipPos.x;
    const relativeY = currentExteriorPos.y - shipPos.y;
    const relativeZ = currentExteriorPos.z - shipPos.z;
    
    console.log('Pre-positioning interior player at relative:', { relativeX, relativeY, relativeZ });
    
    // Set interior player to correct position immediately
    interiorPlayerBody.setTranslation({ x: relativeX, y: relativeY, z: relativeZ }, true);
    interiorPlayerBody.setLinvel({ x: 0, y: 0, z: 0 }, true);
  }
  
  function startExitTransition() {
    if (isTransitioning) return;
    console.log('Starting ship exit transition');
    
    isTransitioning = true;
    transitionProgress = 1;
  }
  
  function updateTransition(deltaTime) {
    const transitionSpeed = 2.0; // Transition duration in seconds
    
    if (!isInsideShip && transitionProgress < 1) {
      // Entering ship
      transitionProgress += deltaTime / transitionSpeed;
      console.log('Entering transition progress:', transitionProgress);
      if (transitionProgress >= 1) {
        transitionProgress = 1;
        finishEnterTransition();
      }
    } else if (isInsideShip && transitionProgress > 0) {
      // Exiting ship
      transitionProgress -= deltaTime / transitionSpeed;
      console.log('Exiting transition progress:', transitionProgress);
      if (transitionProgress <= 0) {
        transitionProgress = 0;
        finishExitTransition();
      }
    }
  }
  
  function finishEnterTransition() {
    console.log('Finishing ship entry');
    isTransitioning = false;
    enterShip();
  }
  
  function finishExitTransition() {
    console.log('Finishing ship exit - calling exitShip()');
    console.log('isInsideShip before exit:', isInsideShip);
    console.log('Player body type before exit:', playerBody.bodyType());
    isTransitioning = false;
    exitShip();
    console.log('isInsideShip after exit:', isInsideShip);
    console.log('Player body type after exit:', playerBody.bodyType());
  }

  function checkStationAutoExit() {
    // Prevent multiple exit calls
    if (isExiting || !isInsideStation) return;
    
    const stationPos = stationPlayerBody.translation();
    const stationVel = stationPlayerBody.linvel();
    const stationLength = 250;
    const wallThickness = 2.0;
    const dockingBayOpeningZ = stationLength / 2 - wallThickness / 2; // Front opening position
    
    // More restrictive exit detection - player must be clearly exiting through docking bay
    const wellBeyondOpening = stationPos.z > dockingBayOpeningZ + 2; // Must be 2 units past the opening
    const strongExitMomentum = stationVel.z > 2.0; // Strong forward movement
    const inDockingBayWidth = Math.abs(stationPos.x) < 30; // Within docking bay width (60/2)
    const inDockingBayHeight = stationPos.y < 30; // Below docking bay ceiling height
    
    if (wellBeyondOpening && strongExitMomentum && inDockingBayWidth && inDockingBayHeight) {
      console.log('STATION EXIT TRIGGERED - Player clearly exiting through docking bay:', { stationPos, stationVel, dockingBayOpeningZ });
      
      // Lock exit to prevent multiple calls
      isExiting = true;
      
      console.log('IMMEDIATE STATION EXIT - NO DELAYS');
      exitStation();
    }
  }

  function checkAutoExit() {
    // Prevent multiple exit calls
    if (isExiting || !isInsideShip) return;
    
    const interiorPos = interiorPlayerBody.translation();
    const interiorVel = interiorPlayerBody.linvel();
    const shipLength = 30;
    const doorZ = shipLength / 2 - 0.4; // Front door position (14.6)
    
    // More restrictive exit detection - player must be clearly exiting
    const wellBeyondDoor = interiorPos.z > doorZ + 1; // Must be 1 unit past the door
    const strongExitMomentum = interiorVel.z > 2.0; // Strong forward movement
    const inDoorWidth = Math.abs(interiorPos.x) < 2; // Within door width
    
    if (wellBeyondDoor && strongExitMomentum && inDoorWidth) {
      console.log('INSTANT EXIT TRIGGERED - Player clearly exiting:', { interiorPos, interiorVel, doorZ });
      
      // Lock exit to prevent multiple calls
      isExiting = true;
      
      console.log('IMMEDIATE EXIT - NO DELAYS');
      
      // Check if ship is docked - if so, exit to station instead of exterior
      if (isShipDocked) {
        console.log('Ship is docked - exiting to station instead of exterior');
        exitShip(); // This function already handles docked ship case
      } else {
        immediateExitToExterior();
      }
    }
  }
  
  function immediateExitToExterior() {
    console.log('Seamless exit - switching from kinematic to dynamic, preserving natural position');
    
    // Get current interior state
    const currentInteriorPos = interiorPlayerBody.translation();
    const currentInteriorVel = interiorPlayerBody.linvel();
    const shipPos = shipBody.translation();
    const shipRot = shipBody.rotation();
    const shipQuat = new THREE.Quaternion(shipRot.x, shipRot.y, shipRot.z, shipRot.w);
    
    // Transform interior position to world coordinates naturally
    const relativePos = new THREE.Vector3(currentInteriorPos.x, currentInteriorPos.y, currentInteriorPos.z);
    relativePos.applyQuaternion(shipQuat);
    
    const exteriorX = shipPos.x + relativePos.x;
    const exteriorY = shipPos.y + relativePos.y;
    const exteriorZ = shipPos.z + relativePos.z;
    
    console.log('Converting kinematic body to dynamic at natural position:', { exteriorX, exteriorY, exteriorZ });
    
    // Create new dynamic body at the actual transformed position
    const dynamicBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(exteriorX, exteriorY, exteriorZ)
      .setCanSleep(false)
      .setLinearDamping(2.0)
      .setAngularDamping(5.0)
      .lockRotations();
    const newDynamicBody = exteriorWorld.createRigidBody(dynamicBodyDesc);
    
    // Create new collider for dynamic body immediately
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const newColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS)
      .setFriction(0.8)
      .setRestitution(0.1);
    const newCollider = exteriorWorld.createCollider(newColliderDesc, newDynamicBody);
    
    console.log('New dynamic body and collider created simultaneously');
    
    // Now remove old kinematic body and collider safely
    if (playerCollider) {
      exteriorWorld.removeCollider(playerCollider, true);
    }
    if (playerBody) {
      exteriorWorld.removeRigidBody(playerBody);
    }
    
    // Switch to new body/collider atomically
    playerBody = newDynamicBody;
    playerCollider = newCollider;
    
    // Transform velocity to world coordinates naturally
    const exitVel = new THREE.Vector3(currentInteriorVel.x, currentInteriorVel.y, currentInteriorVel.z);
    exitVel.applyQuaternion(shipQuat);
    
    // Apply the transformed velocity as-is (let physics handle collisions naturally)
    playerBody.setLinvel({ x: exitVel.x, y: exitVel.y, z: exitVel.z }, true);
    
    // Switch state
    isInsideShip = false;
    isExiting = false;
    
    // Force a physics step to ensure proper collision detection
    exteriorWorld.timestep = 1/60;
    exteriorWorld.step(exteriorEventQueue);
    
    console.log('Seamlessly switched to dynamic body - type:', playerBody.bodyType());
    console.log('Natural exit position:', playerBody.translation());
    console.log('Natural exit velocity:', playerBody.linvel());
    console.log('New collider handle:', newCollider.handle);
  }
  
  function immediateEnterStation() {
    if (isEntering || isInsideStation) return; // Prevent multiple calls and double entry
    isEntering = true;
    console.log('Direct station entry - converting player to station physics');
    
    // Additional safety check - ensure we have valid bodies
    if (!playerBody || !playerCollider || !stationPlayerBody) {
      console.error('Invalid player bodies during station entry');
      isEntering = false;
      return;
    }
    
    // Get current exterior state
    const currentExteriorPos = playerBody.translation();
    const currentExteriorVel = playerBody.linvel();
    const stationPos = stationBody.translation();
    
    // Calculate relative position inside station (exterior pos - station pos)
    const relativeX = currentExteriorPos.x - stationPos.x;
    const relativeY = currentExteriorPos.y - stationPos.y;
    const relativeZ = currentExteriorPos.z - stationPos.z;
    
    console.log('Station entry at relative position:', { relativeX, relativeY, relativeZ });
    
    // Create kinematic body for exterior representation
    const kinematicBodyDesc = RAPIER.RigidBodyDesc.kinematicPositionBased()
      .setTranslation(currentExteriorPos.x, currentExteriorPos.y, currentExteriorPos.z);
    const newKinematicBody = exteriorWorld.createRigidBody(kinematicBodyDesc);
    
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const newColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setSensor(true); // Sensor mode - no collision response
    const newCollider = exteriorWorld.createCollider(newColliderDesc, newKinematicBody);
    
    // Remove old dynamic body safely
    if (playerCollider) {
      exteriorWorld.removeCollider(playerCollider, true);
    }
    if (playerBody) {
      exteriorWorld.removeRigidBody(playerBody);
    }
    
    // Switch to kinematic body
    playerBody = newKinematicBody;
    playerCollider = newCollider;
    isInsideStation = true;
    
    // Set station player to relative position
    stationPlayerBody.setTranslation({ 
      x: relativeX, 
      y: relativeY, 
      z: relativeZ 
    }, true);
    stationPlayerBody.setLinvel({ x: currentExteriorVel.x, y: currentExteriorVel.y, z: currentExteriorVel.z }, true);
    
    console.log('Entered station - station player position:', stationPlayerBody.translation());
    
    // Reset entering flag
    isEntering = false;
  }
  
  function startShipDockingTransition() {
    if (isShipDocked) return;
    console.log('Starting immediate ship docking - seamless like player ship entry');
    
    // Skip transition system - dock immediately like player entry
    smoothDockShip();
  }

  // Ship docking transition functions removed - now using immediate docking like player entry

  function smoothDockShip() {
    if (isShipDocked) return; // Prevent multiple docking calls
    
    console.log('Immediate ship docking - converting exterior ship to kinematic, activating station ship physics');
    
    // Get current ship state directly (no transition data needed)
    const currentShipPos = shipBody.translation();
    const currentShipVel = shipBody.linvel();
    const currentShipRot = shipBody.rotation();
    
    // Store current mesh position to prevent visual glitching
    const currentMeshPos = { x: shipMesh.position.x, y: shipMesh.position.y, z: shipMesh.position.z };
    const currentMeshRot = { x: shipMesh.quaternion.x, y: shipMesh.quaternion.y, z: shipMesh.quaternion.z, w: shipMesh.quaternion.w };
    
    console.log('Using current ship state for immediate docking:', {
      pos: { x: currentShipPos.x.toFixed(2), y: currentShipPos.y.toFixed(2), z: currentShipPos.z.toFixed(2) }
    });
    const stationPos = stationBody.translation();
    const stationRot = stationBody.rotation();
    const stationQuat = new THREE.Quaternion(stationRot.x, stationRot.y, stationRot.z, stationRot.w);
    
    // Calculate relative position inside station (similar to player entry logic)
    const worldRelativePos = new THREE.Vector3(
      currentShipPos.x - stationPos.x,
      currentShipPos.y - stationPos.y, 
      currentShipPos.z - stationPos.z
    );
    
    console.log('Ship position calculation:', {
      shipPos: { x: currentShipPos.x.toFixed(2), y: currentShipPos.y.toFixed(2), z: currentShipPos.z.toFixed(2) },
      stationPos: { x: stationPos.x.toFixed(2), y: stationPos.y.toFixed(2), z: stationPos.z.toFixed(2) },
      worldRelative: { x: worldRelativePos.x.toFixed(2), y: worldRelativePos.y.toFixed(2), z: worldRelativePos.z.toFixed(2) }
    });
    
    // Apply inverse station rotation to get station interior relative position
    const stationQuatInverse = stationQuat.clone().invert();
    worldRelativePos.applyQuaternion(stationQuatInverse);
    
    console.log('Ship entering station at natural interior position (after rotation):', {
      x: worldRelativePos.x.toFixed(2), 
      y: worldRelativePos.y.toFixed(2), 
      z: worldRelativePos.z.toFixed(2)
    });
    console.log('Station interior bounds: Floor Y=1, Ceiling Y=59, Walls X=±99, Z=±124');
    console.log('Ship body type before docking:', shipBody.bodyType());
    
    // Convert exterior ship to kinematic (synchronized by station simulation)
    // First, remove all colliders attached to the ship
    const numColliders = shipBody.numColliders();
    for (let i = numColliders - 1; i >= 0; i--) {
      const collider = shipBody.collider(i);
      if (collider) {
        exteriorWorld.removeCollider(collider, true);
      }
    }
    
    exteriorWorld.removeRigidBody(shipBody);
    
    // Create new kinematic ship body in exterior world
    const kinematicShipBodyDesc = RAPIER.RigidBodyDesc.kinematicPositionBased()
      .setTranslation(currentShipPos.x, currentShipPos.y, currentShipPos.z)
      .setRotation(currentShipRot);
    shipBody = exteriorWorld.createRigidBody(kinematicShipBodyDesc);
    
    // Immediately restore mesh position to prevent visual glitching during body recreation
    shipMesh.position.set(currentMeshPos.x, currentMeshPos.y, currentMeshPos.z);
    shipMesh.quaternion.set(currentMeshRot.x, currentMeshRot.y, currentMeshRot.z, currentMeshRot.w);
    
    // NO COLLIDERS for exterior kinematic ship when docked!
    // All collision detection happens in station world with stationShipBody
    console.log('Ship docked - exterior body is kinematic with NO colliders (station world handles physics)');
    
    // Define ship dimensions for consistency
    const shipWidth = 20, shipHeight = 6, shipLength = 30, wallThickness = 0.4;
    const doorWidth = 4, doorHeight = 4;
    
    // Ship docks at its natural entry position - no artificial constraints
    console.log('Ship docking at natural position:', { 
      x: worldRelativePos.x.toFixed(2), 
      y: worldRelativePos.y.toFixed(2), 
      z: worldRelativePos.z.toFixed(2) 
    });
    console.log('Station docking position adjustment:', {
      naturalY: worldRelativePos.y.toFixed(2),
      naturalZ: worldRelativePos.z.toFixed(2),
      finalY: 'calculated below',
      safeInteriorZ: 'calculated below',
      willCollideWithFloor: worldRelativePos.y < 1.0
    });
    
    // Activate ship physics in station world at natural position
    // Use the natural entry position (worldRelativePos) like player entry does
    // But ensure ship floor doesn't penetrate station floor AND ship is properly inside station
    const shipFloorOffset = -0.4; // Ship floor is 0.4 below ship body
    const stationFloorTop = 0.4; // Station floor top surface
    const minShipY = stationFloorTop - shipFloorOffset + 0.2; // Add small gap to prevent penetration
    
    // Ensure ship is positioned well inside the station to avoid sensor boundary issues
    const stationLength = 250;
    const safeInteriorZ = Math.min(worldRelativePos.z, stationLength/2 - shipLength - 10); // 10 units from back wall
    
    const finalShipY = Math.max(minShipY, worldRelativePos.y);
    
    stationShipBody.setTranslation({ x: worldRelativePos.x, y: finalShipY, z: safeInteriorZ }, true);
    stationShipBody.setRotation(currentShipRot, true);
    
    // Transform velocity to station-relative coordinates
    const shipVel = new THREE.Vector3(currentShipVel.x, currentShipVel.y, currentShipVel.z);
    shipVel.applyQuaternion(stationQuatInverse);
    stationShipBody.setLinvel({ x: shipVel.x, y: shipVel.y, z: shipVel.z }, true);
    
    isShipDocked = true;
    
    console.log('Ship docked smoothly - exterior kinematic, station dynamic');
    console.log('Station ship body position:', stationShipBody.translation());
    console.log('Ship positioned naturally within station bounds');
    
    // Ship positioning debug info
    const shipPos = stationShipBody.translation();
    console.log('Ship positioned in station at:', { x: shipPos.x.toFixed(2), y: shipPos.y.toFixed(2), z: shipPos.z.toFixed(2) });
    
    // Show ship proxy in station interior
    if (stationShipProxy) {
      stationShipProxy.visible = true;
      console.log('Station ship proxy made visible');
    }
    
    // If player is inside ship, they need to know the ship is now docked
    if (isInsideShip) {
      console.log('Player in ship during docking - ship now controlled by station physics');  
    }
    
    // No transition data to clean up - immediate docking complete
  }

  function dockShip() {
    if (isShipDocked) return; // Prevent multiple docking calls
    
    console.log('Ship docking - converting exterior ship to kinematic, activating station ship physics');
    
    // Get current ship state
    const currentShipPos = shipBody.translation();
    const currentShipVel = shipBody.linvel();
    const currentShipRot = shipBody.rotation();
    const stationPos = stationBody.translation();
    
    // Position ship properly within station interior docking bay
    // Station interior coordinates: center at (0,0,0), docking bay at front (positive Z)
    const stationWidth = 200, stationHeight = 60, stationLength = 250;
    const dockingBayDepth = 80;
    
    // Position ship in the middle of the docking bay area
    const relativeX = 0; // Center of docking bay width
    const relativeY = 6; // Above the station floor (floor at Y=1, ship floor at Y=6)  
    const relativeZ = stationLength / 2 - dockingBayDepth / 2; // Middle of docking bay depth
    
    console.log('Ship docking at station interior position:', { relativeX, relativeY, relativeZ });
    console.log('Station interior bounds: Floor Y=1, Ceiling Y=59, Walls X=±99, Z=±124');
    console.log('Ship body type before docking:', shipBody.bodyType());
    
    // Convert exterior ship to kinematic (synchronized by station simulation)
    // First, remove all colliders attached to the ship
    const numColliders = shipBody.numColliders();
    for (let i = numColliders - 1; i >= 0; i--) {
      const collider = shipBody.collider(i);
      if (collider) {
        exteriorWorld.removeCollider(collider, true);
      }
    }
    
    exteriorWorld.removeRigidBody(shipBody);
    
    // Create new kinematic ship body in exterior world
    const kinematicShipBodyDesc = RAPIER.RigidBodyDesc.kinematicPositionBased()
      .setTranslation(currentShipPos.x, currentShipPos.y, currentShipPos.z)
      .setRotation(currentShipRot);
    shipBody = exteriorWorld.createRigidBody(kinematicShipBodyDesc);
    
    // Create sensor colliders for exterior kinematic ship
    const shipWidth = 20, shipHeight = 6, shipLength = 30, wallThickness = 0.4;
    const doorWidth = 4, doorHeight = 4;
    
    // Floor sensor (for general ship detection)
    const shipFloorCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, wallThickness / 2, shipLength / 2)
      .setTranslation(0, wallThickness / 2, 0)
      .setSensor(true);
    exteriorWorld.createCollider(shipFloorCollider, shipBody);
    
    // Door sensor (for player entry) - matching the working exterior ship sensor positioning
    const exteriorDoorSensorCollider = RAPIER.ColliderDesc.cuboid(doorWidth / 2 - 0.5, doorHeight / 2 - 0.5, 0.1)
      .setTranslation(0, doorHeight / 2, shipLength / 2 - wallThickness - 2.0)
      .setSensor(true)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS);
    exteriorWorld.createCollider(exteriorDoorSensorCollider, shipBody);
    
    // Activate ship physics in station world
    stationShipBody.setTranslation({ x: relativeX, y: relativeY, z: relativeZ }, true);
    stationShipBody.setRotation(currentShipRot, true);
    stationShipBody.setLinvel({ x: currentShipVel.x, y: currentShipVel.y, z: currentShipVel.z }, true);
    
    isShipDocked = true;
    
    console.log('Ship docked successfully - exterior kinematic, station dynamic');
    console.log('Station ship body position:', stationShipBody.translation());
    console.log('Station interior bounds: Floor Y=1, Ceiling Y=59, Left X=-99, Right X=99, Front Z=124, Back Z=-124');
    console.log('Ship should be within these bounds to collide with station walls/floors');
    
    // Show ship proxy in station interior
    if (stationShipProxy) {
      stationShipProxy.visible = true;
      console.log('Station ship proxy made visible');
    }
    
    // If player is inside ship, they need to know the ship is now docked
    if (isInsideShip) {
      console.log('Player in ship during docking - ship now controlled by station physics');  
    }
  }
  
  function undockShip() {
    if (!isShipDocked) return;
    
    console.log('Undocking ship - converting back to dynamic');
    
    // Get current ship state from both exterior and station worlds
    const currentShipPos = shipBody.translation();
    const currentShipRot = shipBody.rotation();
    const stationShipVel = stationShipBody.translation() ? stationShipBody.linvel() : { x: 0, y: 0, z: 0 };
    const stationShipAngVel = stationShipBody.translation() ? stationShipBody.angvel() : { x: 0, y: 0, z: 0 };
    
    console.log('Undocking ship with velocity:', stationShipVel);
    
    // Remove kinematic body and its colliders
    const numColliders = shipBody.numColliders();
    for (let i = numColliders - 1; i >= 0; i--) {
      const collider = shipBody.collider(i);
      if (collider) {
        exteriorWorld.removeCollider(collider, true);
      }
    }
    
    exteriorWorld.removeRigidBody(shipBody);
    
    // Create new dynamic ship body
    const dynamicShipBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(currentShipPos.x, currentShipPos.y, currentShipPos.z)
      .setRotation(currentShipRot)
      .setCanSleep(false)
      .setLinearDamping(0.5)
      .setAngularDamping(2.0);
    shipBody = exteriorWorld.createRigidBody(dynamicShipBodyDesc);
    
    // Apply momentum from station world to maintain smooth transition
    shipBody.setLinvel({ x: stationShipVel.x, y: stationShipVel.y, z: stationShipVel.z }, true);
    shipBody.setAngvel({ x: stationShipAngVel.x, y: stationShipAngVel.y, z: stationShipAngVel.z }, true);
    
    console.log('Applied momentum to undocked ship:', stationShipVel);
    
    // Recreate full ship colliders (not sensors) - same as original ship creation
    const shipWidth = 20, shipHeight = 6, shipLength = 30, wallThickness = 0.4;
    const doorWidth = 4, doorHeight = 4;
    
    // Ship floor
    const shipFloorCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, wallThickness / 2, shipLength / 2)
      .setTranslation(0, wallThickness / 2, 0);
    exteriorWorld.createCollider(shipFloorCollider, shipBody);
    
    // Ship ceiling
    const shipCeilingCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, wallThickness / 2, shipLength / 2)
      .setTranslation(0, shipHeight - wallThickness / 2, 0);
    exteriorWorld.createCollider(shipCeilingCollider, shipBody);
    
    // Ship walls
    const shipLeftWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, shipHeight / 2, shipLength / 2)
      .setTranslation(-shipWidth / 2 + wallThickness / 2, shipHeight / 2, 0);
    exteriorWorld.createCollider(shipLeftWallCollider, shipBody);
    
    const shipRightWallCollider = RAPIER.ColliderDesc.cuboid(wallThickness / 2, shipHeight / 2, shipLength / 2)
      .setTranslation(shipWidth / 2 - wallThickness / 2, shipHeight / 2, 0);
    exteriorWorld.createCollider(shipRightWallCollider, shipBody);
    
    const shipBackWallCollider = RAPIER.ColliderDesc.cuboid(shipWidth / 2, shipHeight / 2, wallThickness / 2)
      .setTranslation(0, shipHeight / 2, -shipLength / 2 + wallThickness / 2);
    exteriorWorld.createCollider(shipBackWallCollider, shipBody);
    
    // Ship front walls with door opening
    const frontWallLeftCollider = RAPIER.ColliderDesc.cuboid((shipWidth - doorWidth) / 4, shipHeight / 2, wallThickness / 2)
      .setTranslation(-(shipWidth + doorWidth) / 4, shipHeight / 2, shipLength / 2 - wallThickness / 2);
    exteriorWorld.createCollider(frontWallLeftCollider, shipBody);
    
    const frontWallRightCollider = RAPIER.ColliderDesc.cuboid((shipWidth - doorWidth) / 4, shipHeight / 2, wallThickness / 2)
      .setTranslation((shipWidth + doorWidth) / 4, shipHeight / 2, shipLength / 2 - wallThickness / 2);
    exteriorWorld.createCollider(frontWallRightCollider, shipBody);
    
    const frontWallTopCollider = RAPIER.ColliderDesc.cuboid(doorWidth / 2, (shipHeight - doorHeight) / 2, wallThickness / 2)
      .setTranslation(0, shipHeight - (shipHeight - doorHeight) / 2, shipLength / 2 - wallThickness / 2);
    exteriorWorld.createCollider(frontWallTopCollider, shipBody);

    // Create exterior door sensor for ship entry detection
    const exteriorDoorSensorCollider = RAPIER.ColliderDesc.cuboid(doorWidth / 2 - 0.5, doorHeight / 2 - 0.5, 0.1)
      .setTranslation(0, doorHeight / 2, shipLength / 2 - wallThickness - 2.0)
      .setSensor(true)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS);
    exteriorWorld.createCollider(exteriorDoorSensorCollider, shipBody);
    
    isShipDocked = false;
    
    // Deactivate ship physics in station world by moving it out of the way
    stationShipBody.setTranslation({ x: 0, y: -300, z: 0 }, true);
    stationShipBody.setLinvel({ x: 0, y: 0, z: 0 }, true);
    stationShipBody.setAngvel({ x: 0, y: 0, z: 0 }, true);
    
    // Hide ship proxy in station interior
    if (stationShipProxy) {
      stationShipProxy.visible = false;
      console.log('Station ship proxy hidden');
    }
    
    // Set undock cooldown to prevent immediate re-docking
    lastUndockTime = Date.now();
    
    console.log('Ship undocked successfully - body type:', shipBody.bodyType());
    console.log('Ship returned to exterior world physics with momentum preserved');
    console.log('Undock cooldown set - ship cannot re-dock for 2 seconds');
  }

  function exitShip() {
    console.log('Seamlessly exiting ship');
    console.log('Ship docking status check: isShipDocked =', isShipDocked);
    console.log('Ship position:', shipBody.translation());
    if (stationShipBody) {
      console.log('Station ship body position:', stationShipBody.translation());
    }
    
    isInsideShip = false;
    
    // Get current interior player state
    const currentInteriorPos = interiorPlayerBody.translation();
    const currentInteriorVel = interiorPlayerBody.linvel();
    
    // Determine exit context based on ship's current position, not just docking status
    // If ship is inside the station docking bay bounds, player should exit to station interior
    const shipPos = shipBody.translation();
    const stationPos = stationBody.translation();
    
    // Check if ship is within station bounds (more reliable than docking status)
    const relativeToStation = {
      x: shipPos.x - stationPos.x,
      y: shipPos.y - stationPos.y,
      z: shipPos.z - stationPos.z
    };
    
    // Station interior bounds: ±99 X, 1-59 Y, ±124 Z
    const isShipInsideStation = Math.abs(relativeToStation.x) < 90 && 
                                relativeToStation.y > -5 && relativeToStation.y < 65 &&
                                Math.abs(relativeToStation.z) < 120;
    
    console.log('Ship position context check:');
    console.log('  Ship world pos:', shipPos);
    console.log('  Station world pos:', stationPos);
    console.log('  Ship relative to station:', relativeToStation);
    console.log('  Is ship inside station bounds:', isShipInsideStation);
    console.log('  Docking status (isShipDocked):', isShipDocked);
    
    // Use position context instead of docking status for exit decision
    if (isShipInsideStation) {
      console.log('Ship is inside station - player entering station interior');
      isInsideStation = true;
      
      // Ensure ship physics is active in station world if not already
      if (!isShipDocked) {
        console.log('Ship was inside station but not marked as docked - activating station physics');
        // Quick dock the ship to activate station physics
        const stationRot = stationBody.rotation();
        const stationQuat = new THREE.Quaternion(stationRot.x, stationRot.y, stationRot.z, stationRot.w);
        const stationQuatInverse = stationQuat.clone().invert();
        
        // Convert ship position to station-relative coordinates
        const worldRelativePos = new THREE.Vector3(
          relativeToStation.x, relativeToStation.y, relativeToStation.z
        );
        worldRelativePos.applyQuaternion(stationQuatInverse);
        
        // Activate ship physics in station world
        stationShipBody.setTranslation({ x: worldRelativePos.x, y: worldRelativePos.y, z: worldRelativePos.z }, true);
        stationShipBody.setRotation(shipBody.rotation(), true);
        stationShipBody.setLinvel(shipBody.linvel(), true);
        
        isShipDocked = true;
        console.log('Ship physics activated in station world at:', worldRelativePos);
        
        // Show ship proxy in station interior
        if (stationShipProxy) {
          stationShipProxy.visible = true;
          console.log('Station ship proxy made visible');
        }
      }
      
      // Get station physics body position
      const stationShipPos = stationShipBody.translation();
      const stationShipRot = stationShipBody.rotation();
      const shipQuat = new THREE.Quaternion(stationShipRot.x, stationShipRot.y, stationShipRot.z, stationShipRot.w);
      
      // Transform player position from ship interior to station interior
      const relativePos = new THREE.Vector3(currentInteriorPos.x, currentInteriorPos.y, currentInteriorPos.z);
      relativePos.applyQuaternion(shipQuat);
      
      // Position player in station relative to docked ship
      // Place player on station floor (Y=1) near ship door, not at ship's Y position
      const stationFloorY = 1.5; // Slightly above station floor to prevent clipping
      const stationPlayerPos = {
        x: stationShipPos.x + relativePos.x,
        y: stationFloorY, // Always place on station floor, not ship height
        z: stationShipPos.z + relativePos.z
      };
      
      // Set station player body position
      stationPlayerBody.setTranslation(stationPlayerPos, true);
      stationPlayerBody.setLinvel({ x: currentInteriorVel.x, y: currentInteriorVel.y, z: currentInteriorVel.z }, true);
      
      console.log('Player entered station at:', stationPlayerPos);
      
      // Update exterior player body to match station position to prevent collision re-entry
      // Convert station-relative position back to world coordinates
      const stationWorldPos = stationBody.translation();
      const exteriorPlayerPos = {
        x: stationWorldPos.x + stationPlayerPos.x,
        y: stationWorldPos.y + stationPlayerPos.y,
        z: stationWorldPos.z + stationPlayerPos.z
      };
      
      // Update exterior kinematic player body to match
      if (playerBody) {
        playerBody.setTranslation(exteriorPlayerPos, true);
        console.log('Updated exterior player body to match station position:', exteriorPlayerPos);
      }
      
      // Hide interior player mesh
      interiorPlayerBody.setTranslation({ x: 0, y: -100, z: 0 }, true);
      interiorPlayerBody.setLinvel({ x: 0, y: 0, z: 0 }, true);
      
      // Reset exiting flag - critical for allowing ship re-entry in station
      isExiting = false;
      
      return; // Exit early - we're in station now
    }
    
    // Not docked - normal exit to exterior
    console.log('Exiting ship to exterior');
    const currentShipPos = shipBody.translation();
    
    // Calculate exterior world position with ship rotation
    const shipRot = shipBody.rotation();
    const shipQuat = new THREE.Quaternion(shipRot.x, shipRot.y, shipRot.z, shipRot.w);
    
    // Apply ship rotation to interior relative position
    const relativePos = new THREE.Vector3(currentInteriorPos.x, currentInteriorPos.y, currentInteriorPos.z);
    relativePos.applyQuaternion(shipQuat);
    
    // Exit player at a safe position outside the ship (front door area)
    const safeExitOffset = new THREE.Vector3(0, 1, 18); // In front of ship, above ground
    safeExitOffset.applyQuaternion(shipQuat);
    
    const exteriorX = currentShipPos.x + safeExitOffset.x;
    const exteriorY = Math.max(currentShipPos.y + safeExitOffset.y, 2); // Ensure above ground
    const exteriorZ = currentShipPos.z + safeExitOffset.z;
    
    console.log('Switching control - exterior position:', { exteriorX, exteriorY, exteriorZ });
    
    // Convert kinematic player back to dynamic body at current position
    console.log('Removing kinematic body...');
    if (playerCollider) {
      exteriorWorld.removeCollider(playerCollider, true);
    }
    if (playerBody) {
      exteriorWorld.removeRigidBody(playerBody);
    }
    
    console.log('Creating new dynamic body at:', { exteriorX, exteriorY, exteriorZ });
    const dynamicPlayerBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(exteriorX, exteriorY, exteriorZ)
      .setCanSleep(false)
      .setLinearDamping(2.0)
      .setAngularDamping(5.0)
      .lockRotations();
    playerBody = exteriorWorld.createRigidBody(dynamicPlayerBodyDesc);
    
    console.log('New dynamic body created with handle:', playerBody.handle);
    console.log('Body type is now:', playerBody.bodyType());
    
    // Maintain velocity from interior simulation
    playerBody.setLinvel({ x: currentInteriorVel.x, y: currentInteriorVel.y, z: currentInteriorVel.z }, true);
    
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const dynamicPlayerColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS)
      .setFriction(0.8)
      .setRestitution(0.1);
    playerCollider = exteriorWorld.createCollider(dynamicPlayerColliderDesc, playerBody);
    
    console.log('Exterior player taking control at:', playerBody.translation());
    console.log('Player body type:', playerBody.bodyType());
    console.log('Player body handle:', playerBody.handle);
    console.log('Player collider created:', playerCollider ? 'Yes' : 'No');
    console.log('Player collider handle:', playerCollider.handle);
    
    // Verify body is properly added to world
    console.log('Exterior world bodies count:', exteriorWorld.bodies.len());
    console.log('Exterior world colliders count:', exteriorWorld.colliders.len());
    
    // Force a small upward impulse to test collision
    playerBody.applyImpulse({ x: 0, y: 1, z: 0 }, true);
    
    // Test that player body is responsive
    setTimeout(() => {
      console.log('Player position after 100ms:', playerBody.translation());
      console.log('Player velocity:', playerBody.linvel());
      console.log('Player is grounded:', isGrounded);
    }, 100);
  }
  
  function enterShipFromStation() {
    if (isEntering || isInsideShip) return; // Prevent multiple calls and double entry
    isEntering = true;
    console.log('🚀 ENTERING SHIP FROM STATION - NESTED STATIC PROXY TRANSITION');
    console.log('  Before: isInsideStation =', isInsideStation, ', isInsideShip =', isInsideShip);
    isInsideStation = false;
    isInsideShip = true;
    console.log('  After: isInsideStation =', isInsideStation, ', isInsideShip =', isInsideShip);
    console.log('  Should now render ship interior view only');
    
    // Get current station player state
    const currentStationPos = stationPlayerBody.translation();
    const currentStationVel = stationPlayerBody.linvel();
    
    // Get ship position in station world
    const stationShipPos = stationShipBody.translation();
    const stationShipRot = stationShipBody.rotation();
    const shipQuat = new THREE.Quaternion(stationShipRot.x, stationShipRot.y, stationShipRot.z, stationShipRot.w);
    
    // Calculate player position relative to ship
    const relativePos = new THREE.Vector3(
      currentStationPos.x - stationShipPos.x,
      currentStationPos.y - stationShipPos.y,
      currentStationPos.z - stationShipPos.z
    );
    
    // Apply inverse ship rotation to get ship-local position
    relativePos.applyQuaternion(shipQuat.clone().invert());
    
    // Set interior player position
    interiorPlayerBody.setTranslation({
      x: relativePos.x,
      y: relativePos.y,
      z: relativePos.z
    }, true);
    interiorPlayerBody.setLinvel({ x: currentStationVel.x, y: currentStationVel.y, z: currentStationVel.z }, true);
    
    // Convert exterior player body to kinematic (same as normal ship entry)
    // Remove existing dynamic body and collider
    if (playerCollider) {
      exteriorWorld.removeCollider(playerCollider, true);
    }
    if (playerBody) {
      exteriorWorld.removeRigidBody(playerBody);
    }
    
    // Get current exterior ship position for kinematic body
    const currentShipPos = shipBody.translation();
    const currentShipRot = shipBody.rotation();
    
    // Create new kinematic body at ship position (player follows ship)
    const kinematicBodyDesc = RAPIER.RigidBodyDesc.kinematicPositionBased()
      .setTranslation(currentShipPos.x, currentShipPos.y, currentShipPos.z);
    const newKinematicBody = exteriorWorld.createRigidBody(kinematicBodyDesc);
    
    // Create sensor collider for kinematic body
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const newColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setSensor(true);
    const newCollider = exteriorWorld.createCollider(newColliderDesc, newKinematicBody);
    
    // Switch to new kinematic body (same as normal ship entry)
    playerBody = newKinematicBody;
    playerCollider = newCollider;
    
    console.log('Exterior player body converted to kinematic and follows ship');
    
    // Hide station player
    stationPlayerBody.setTranslation({ x: 0, y: -200, z: 0 }, true);
    stationPlayerBody.setLinvel({ x: 0, y: 0, z: 0 }, true);
    
    console.log('Player entered ship from station at interior position:', relativePos);
    console.log('Player now in ship interior world, exterior body follows ship kinematically');
    
    // Reset entering flag
    isEntering = false;
  }
  
  function exitStation() {
    console.log('Seamlessly exiting station');
    isInsideStation = false;
    
    // Get current station player state
    const currentStationPos = stationPlayerBody.translation();
    const currentStationVel = stationPlayerBody.linvel();
    const stationPos = stationBody.translation();
    
    // Calculate exterior world position
    const stationRot = stationBody.rotation();
    const stationQuat = new THREE.Quaternion(stationRot.x, stationRot.y, stationRot.z, stationRot.w);
    
    // Apply station rotation to interior relative position
    const relativePos = new THREE.Vector3(currentStationPos.x, currentStationPos.y, currentStationPos.z);
    relativePos.applyQuaternion(stationQuat);
    
    // Exit player at a safe position outside the station  
    const safeExitOffset = new THREE.Vector3(-15, 1, 0); // Left side of station, just above ground
    safeExitOffset.applyQuaternion(stationQuat);
    
    const exteriorX = stationPos.x + safeExitOffset.x;
    const exteriorY = Math.max(stationPos.y + safeExitOffset.y, 2); // Ensure above ground (ground at -0.5)
    const exteriorZ = stationPos.z + safeExitOffset.z;
    
    console.log('Switching control - exterior position:', { exteriorX, exteriorY, exteriorZ });
    
    // Convert kinematic player back to dynamic body at current position (exact ship exit pattern)
    console.log('Removing kinematic body...');
    if (playerCollider) {
      exteriorWorld.removeCollider(playerCollider, true);
    }
    if (playerBody) {
      exteriorWorld.removeRigidBody(playerBody);
    }
    
    console.log('Creating new dynamic body at:', { exteriorX, exteriorY, exteriorZ });
    const dynamicPlayerBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(exteriorX, exteriorY, exteriorZ)
      .setCanSleep(false)
      .setLinearDamping(2.0)
      .setAngularDamping(5.0)
      .lockRotations();
    playerBody = exteriorWorld.createRigidBody(dynamicPlayerBodyDesc);
    
    console.log('New dynamic body created with handle:', playerBody.handle);
    console.log('Body type is now:', playerBody.bodyType());
    
    // Maintain velocity from station interior simulation
    playerBody.setLinvel({ x: currentStationVel.x, y: currentStationVel.y, z: currentStationVel.z }, true);
    
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const dynamicPlayerColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS)
      .setFriction(0.8)
      .setRestitution(0.1);
    playerCollider = exteriorWorld.createCollider(dynamicPlayerColliderDesc, playerBody);
    
    console.log('Exterior player taking control at:', playerBody.translation());
    console.log('Player body type:', playerBody.bodyType());
    console.log('Player body handle:', playerBody.handle);
    console.log('Player collider created:', playerCollider ? 'Yes' : 'No');
    console.log('Player collider handle:', playerCollider.handle);
    
    // Verify body is properly added to world
    console.log('Exterior world bodies count:', exteriorWorld.bodies.len());
    console.log('Exterior world colliders count:', exteriorWorld.colliders.len());
    
    // Force a small upward impulse to test collision (exact ship exit pattern)
    playerBody.applyImpulse({ x: 0, y: 1, z: 0 }, true);
    
    // Move station player out of the way
    stationPlayerBody.setTranslation({ x: 0, y: -200, z: 0 }, true);
    stationPlayerBody.setLinvel({ x: 0, y: 0, z: 0 }, true);
    
    // Set cooldown to prevent immediate station re-entry after exiting
    lastShipExitToStationTime = Date.now();
    console.log('Station exit cooldown set - preventing re-entry for 1 second');
    
    // Test that player body is responsive
    setTimeout(() => {
      console.log('Player position after 100ms:', playerBody.translation());
      console.log('Player velocity:', playerBody.linvel());
      console.log('Player is grounded:', isGrounded);
    }, 100);
    
    // If ship is still docked when player exits station, undock it
    if (isShipDocked) {
      console.log('Player exiting station - undocking ship as well');
      undockShip();
    }
    
    // Reset exiting flag
    isExiting = false;
  }
  
  function checkGrounded() {
    try {
      if (isInsideShip && interiorPlayerBody) {
        const playerPos = interiorPlayerBody.translation();
        if (!playerPos) return;
        
        const rayStart = { x: playerPos.x, y: playerPos.y - 0.65, z: playerPos.z };
        const rayDir = { x: 0, y: -1, z: 0 };
        const rayLength = 0.3;
        
        const ray = new RAPIER.Ray(rayStart, rayDir);
        const hit = interiorWorld.castRay(ray, rayLength, true);
        
        isGrounded = hit !== null;
        
        // Debug interior physics (only every 60 frames to avoid spam)
        if (false && Math.random() < 0.016) { // Disabled debug logging
          console.log('Interior raycast:', {
            playerPos,
            rayStart,
            rayLength,
            hit: hit !== null,
            hitDistance: hit ? hit.toi : null,
            numBodies: interiorWorld.bodies.len(),
            numColliders: interiorWorld.colliders.len(),
            playerVelocity: interiorPlayerBody.linvel()
          });
        }
      } else if (!isInsideShip && playerBody) {
        const playerPos = playerBody.translation();
        if (!playerPos) return;
        
        const rayStart = { x: playerPos.x, y: playerPos.y - 0.65, z: playerPos.z };
        const rayDir = { x: 0, y: -1, z: 0 };
        const rayLength = 0.3;
        
        const ray = new RAPIER.Ray(rayStart, rayDir);
        const hit = exteriorWorld.castRay(ray, rayLength, true);
        
        isGrounded = hit !== null;
      }
    } catch (error) {
      console.warn('Error checking grounded state:', error);
      isGrounded = false; // Safe fallback
    }
  }
  
  function updatePlayer(deltaTime) {
    const groundSpeed = 4;
    const airSpeed = 1.5; // Slower movement in air
    const jumpSpeed = 7;
    
    try {
      checkGrounded();
      
      let velocity, currentPlayerBody;
      
      if (isInsideShip) {
        currentPlayerBody = interiorPlayerBody;
        velocity = interiorPlayerBody.linvel();
      } else if (isInsideStation) {
        currentPlayerBody = stationPlayerBody;
        velocity = stationPlayerBody.linvel();
      } else {
        currentPlayerBody = playerBody;
        velocity = playerBody.linvel();
      }
      
      // Safety check - make sure we have valid physics bodies
      if (!currentPlayerBody || !velocity) {
        console.warn('Invalid player body or velocity, skipping update');
        return;
      }
    
    // Reset canJump when grounded
    if (isGrounded && velocity.y < 1.0) {
      canJump = true;
    }
    
    let moveX = 0;
    let moveZ = 0;
    
    if (movement.forward) moveZ -= 1;
    if (movement.backward) moveZ += 1;
    if (movement.left) moveX -= 1;
    if (movement.right) moveX += 1;
    
    if (moveX !== 0 || moveZ !== 0) {
      const length = Math.sqrt(moveX * moveX + moveZ * moveZ);
      moveX /= length;
      moveZ /= length;
    }
    
    // Use different speed based on grounded state
    const currentSpeed = isGrounded ? groundSpeed : airSpeed;
    const impulse = { x: moveX * currentSpeed * deltaTime * 30, y: 0, z: moveZ * currentSpeed * deltaTime * 30 };
    
    // Only allow jump when grounded, can jump, and not already moving upward significantly
    if (movement.jump && isGrounded && canJump && velocity.y < 1.0) {
      impulse.y = jumpSpeed;
      canJump = false; // Prevent multiple jumps until grounded again
      movement.jump = false; // Reset jump to prevent continuous jumping
    }
    
    currentPlayerBody.applyImpulse(impulse, true);
    
    const pos = currentPlayerBody.translation();
    
    // Update player mesh position and sync kinematic body if inside ship
    if (isInsideShip) {
      // Only show debug player mesh if interior body is at a reasonable position (not at init position)
      if (pos.y > -50) { // Interior body was initialized at y: -100, so wait until properly positioned
        debugPlayerMesh.position.set(pos.x, pos.y, pos.z);
        debugPlayerMesh.visible = true;
      } else {
        debugPlayerMesh.visible = false;
      }
      
      // Sync exterior kinematic body with interior dynamic body position
      // Transform interior position to exterior world coordinates with ship rotation
      const shipPos = shipBody.translation();
      const shipRot = shipBody.rotation();
      const shipQuat = new THREE.Quaternion(shipRot.x, shipRot.y, shipRot.z, shipRot.w);
      
      // Apply ship rotation to interior relative position
      const relativePos = new THREE.Vector3(pos.x, pos.y, pos.z);
      relativePos.applyQuaternion(shipQuat);
      
      const exteriorPos = {
        x: shipPos.x + relativePos.x,
        y: shipPos.y + relativePos.y,
        z: shipPos.z + relativePos.z
      };
      
      playerBody.setNextKinematicTranslation(exteriorPos);
      playerBody.setNextKinematicRotation(shipRot);
      
      // Position and rotate player mesh in real world interior (translated from static proxy)
      playerMesh.position.set(exteriorPos.x, exteriorPos.y, exteriorPos.z);
      playerMesh.quaternion.set(shipRot.x, shipRot.y, shipRot.z, shipRot.w);
      
      // Interior camera follows player with mouse look
      if (isFirstPerson) {
        // First person interior view - camera at proxy player position with mouse look
        // Camera positioned at player head height (1.5 units above player center)
        interiorCamera.position.set(pos.x, pos.y + 1.5, pos.z);
        
        // Apply mouse look rotation directly in interior space (not affected by ship rotation)
        // This gives proper FPS control regardless of ship orientation
        const mouseLookEuler = new THREE.Euler(cameraRotation.pitch, cameraRotation.yaw, 0, 'YXZ');
        const mouseLookQuat = new THREE.Quaternion().setFromEuler(mouseLookEuler);
        interiorCamera.quaternion.copy(mouseLookQuat);
      } else {
        // Third person interior view - fixed camera (zoomed out for better overview)
        interiorCamera.position.set(0, 6, 18);
        interiorCamera.lookAt(0, 1.5, 0);
      }
      
      // Exterior camera follows real world player representation when inside ship
      if (isFirstPerson) {
        // First person - camera at player head position with ship-relative orientation
        // Position camera at head height relative to ship's up direction
        const shipQuat = new THREE.Quaternion(shipRot.x, shipRot.y, shipRot.z, shipRot.w);
        const shipUpOffset = new THREE.Vector3(0, 1.5, 0).applyQuaternion(shipQuat);
        exteriorCamera.position.set(
          exteriorPos.x + shipUpOffset.x, 
          exteriorPos.y + shipUpOffset.y, 
          exteriorPos.z + shipUpOffset.z
        );
        
        // Camera should be oriented relative to ship's coordinate system
        // Player feels upright relative to ship floor, not world floor
        const mouseLookEuler = new THREE.Euler(cameraRotation.pitch, cameraRotation.yaw, 0, 'YXZ');
        const mouseLookQuat = new THREE.Quaternion().setFromEuler(mouseLookEuler);
        
        // Combine ship rotation with mouse look: ship orientation first, then mouse look
        const combinedQuat = shipQuat.clone().multiply(mouseLookQuat);
        exteriorCamera.quaternion.copy(combinedQuat);
      } else {
        // Third person - camera much further behind and above player
        exteriorCamera.position.set(exteriorPos.x, exteriorPos.y + 15, exteriorPos.z + 30);
        exteriorCamera.lookAt(exteriorPos.x, exteriorPos.y + 1, exteriorPos.z);
      }
    } else if (isInsideStation) {
      // Player is inside station
      // Show debug station player mesh if positioned correctly
      if (pos.y > -150) { // Station body was initialized at y: -200
        debugStationPlayerMesh.position.set(pos.x, pos.y, pos.z);
        debugStationPlayerMesh.visible = true;
      } else {
        debugStationPlayerMesh.visible = false;
      }
      
      // Sync exterior kinematic body with station interior dynamic body position
      // Transform station interior position to exterior world coordinates
      const stationPos = stationBody.translation();
      const stationRot = stationBody.rotation();
      const stationQuat = new THREE.Quaternion(stationRot.x, stationRot.y, stationRot.z, stationRot.w);
      
      // Apply station rotation to interior relative position
      const relativePos = new THREE.Vector3(pos.x, pos.y, pos.z);
      relativePos.applyQuaternion(stationQuat);
      
      const exteriorPos = {
        x: stationPos.x + relativePos.x,
        y: stationPos.y + relativePos.y,
        z: stationPos.z + relativePos.z
      };
      
      playerBody.setNextKinematicTranslation(exteriorPos);
      playerBody.setNextKinematicRotation(stationRot);
      
      // Position and rotate player mesh in real world
      playerMesh.position.set(exteriorPos.x, exteriorPos.y, exteriorPos.z);
      playerMesh.quaternion.set(stationRot.x, stationRot.y, stationRot.z, stationRot.w);
      
      // Station camera follows player with mouse look
      if (isFirstPerson) {
        // First person station interior view - camera at proxy player position with mouse look
        stationCamera.position.set(pos.x, pos.y + 1.5, pos.z);
        
        // Apply mouse look rotation directly in station interior space
        const mouseLookEuler = new THREE.Euler(cameraRotation.pitch, cameraRotation.yaw, 0, 'YXZ');
        const mouseLookQuat = new THREE.Quaternion().setFromEuler(mouseLookEuler);
        stationCamera.quaternion.copy(mouseLookQuat);
      } else {
        // Third person station interior view - fixed camera (further back for large station)
        stationCamera.position.set(0, 40, 100);
        stationCamera.lookAt(0, 10, 0);
      }
      
      // Exterior camera follows real world player representation when inside station
      if (isFirstPerson) {
        // First person - camera at player head position with station-relative orientation
        const stationUpOffset = new THREE.Vector3(0, 1.5, 0).applyQuaternion(stationQuat);
        exteriorCamera.position.set(
          exteriorPos.x + stationUpOffset.x, 
          exteriorPos.y + stationUpOffset.y, 
          exteriorPos.z + stationUpOffset.z
        );
        
        // Camera should be oriented relative to station's coordinate system
        const mouseLookEuler = new THREE.Euler(cameraRotation.pitch, cameraRotation.yaw, 0, 'YXZ');
        const mouseLookQuat = new THREE.Quaternion().setFromEuler(mouseLookEuler);
        
        // Combine station rotation with mouse look
        const combinedQuat = stationQuat.clone().multiply(mouseLookQuat);
        exteriorCamera.quaternion.copy(combinedQuat);
      } else {
        // Third person - camera behind and above player
        exteriorCamera.position.set(exteriorPos.x, exteriorPos.y + 15, exteriorPos.z + 30);
        exteriorCamera.lookAt(exteriorPos.x, exteriorPos.y + 1, exteriorPos.z);
      }
    } else {
      // Position player mesh for exterior view
      playerMesh.position.set(pos.x, pos.y, pos.z);
      
      // Hide debug player mesh when outside
      debugPlayerMesh.visible = false;
      
      // Update cameras - exterior camera follows player position
      if (isFirstPerson) {
        // First person - camera at player head position with mouse look
        exteriorCamera.position.set(pos.x, pos.y + 1.5, pos.z);
        
        // Apply mouse look rotation directly (no ship rotation interference outside)
        const mouseLookEuler = new THREE.Euler(cameraRotation.pitch, cameraRotation.yaw, 0, 'YXZ');
        const mouseLookQuat = new THREE.Quaternion().setFromEuler(mouseLookEuler);
        exteriorCamera.quaternion.copy(mouseLookQuat);
      } else {
        // Third person - camera much further behind and above player  
        exteriorCamera.position.set(pos.x, pos.y + 15, pos.z + 30);
        exteriorCamera.lookAt(pos.x, pos.y + 1, pos.z);
      }
    }
    
    // Update debug states
    playerPosition = { x: Math.round(pos.x * 100) / 100, y: Math.round(pos.y * 100) / 100, z: Math.round(pos.z * 100) / 100 };
    playerVelocity = { x: Math.round(velocity.x * 100) / 100, y: Math.round(velocity.y * 100) / 100, z: Math.round(velocity.z * 100) / 100 };
    
    } catch (error) {
      console.warn('Error updating player:', error);
    }
  }
  
  
  function updateShip(deltaTime) {
    const thrustPower = 2000; // Much stronger thrust power
    const liftPower = 2500; // Much stronger lift power  
    const torquePower = 1000; // Much stronger torque power
    
    try {
    
    // If ship is docked, use station physics instead of exterior physics
    if (isShipDocked) {
      // Always update exterior kinematic ship to match station physics
      const stationShipPos = stationShipBody.translation();
      const stationShipRot = stationShipBody.rotation();
      const stationPos = stationBody.translation();
      const stationRot = stationBody.rotation();
      
      // Debug logging (occasionally)
      if (Math.random() < 0.01) { // 1% chance per frame
        console.log('Ship in station world at:', stationShipPos);
        console.log('Station world bounds - Floor at Y=1, Ceiling at Y=59, Walls at X=±99, Z=±124');
      }
      
      // Convert station-relative position to world position (same logic as player sync)
      const stationQuat = new THREE.Quaternion(stationRot.x, stationRot.y, stationRot.z, stationRot.w);
      
      // Apply station rotation to ship's relative position
      const relativeShipPos = new THREE.Vector3(stationShipPos.x, stationShipPos.y, stationShipPos.z);
      relativeShipPos.applyQuaternion(stationQuat);
      
      const worldShipPos = {
        x: stationPos.x + relativeShipPos.x,
        y: stationPos.y + relativeShipPos.y,
        z: stationPos.z + relativeShipPos.z
      };
      
      // Update exterior kinematic ship position
      shipBody.setTranslation(worldShipPos, true);
      shipBody.setRotation(stationShipRot, true);
      
      // Update ship mesh to match
      shipMesh.position.set(worldShipPos.x, worldShipPos.y, worldShipPos.z);
      shipMesh.quaternion.set(stationShipRot.x, stationShipRot.y, stationShipRot.z, stationShipRot.w);
      
      // If player is trying to move the ship while docked, apply forces to station physics body
      const isMovingShip = shipMovement.forward || shipMovement.backward || 
                          shipMovement.left || shipMovement.right || 
                          shipMovement.up || shipMovement.down;
      
      if (isMovingShip && isInsideShip) {
        console.log('Applying ship movement forces to station physics body');
        
        // Use station ship body for physics simulation
        const stationShipRotation = stationShipBody.rotation();
        const stationShipQuat = new THREE.Quaternion(stationShipRotation.x, stationShipRotation.y, stationShipRotation.z, stationShipRotation.w);
        
        // Forward/backward thrust in ship's local direction
        let thrustZ = 0;
        if (shipMovement.forward) {
          thrustZ -= 1;
        }
        if (shipMovement.backward) {
          thrustZ += 1;
        }
        
        if (thrustZ !== 0) {
          const forwardVector = new THREE.Vector3(0, 0, thrustZ).applyQuaternion(stationShipQuat);
          const thrustForce = {
            x: forwardVector.x * thrustPower,
            y: forwardVector.y * thrustPower,
            z: forwardVector.z * thrustPower
          };
          stationShipBody.addForce(thrustForce, true);
        }
        
        // Vertical movement
        let verticalThrust = 0;
        if (shipMovement.up) {
          verticalThrust += 1;
        }
        if (shipMovement.down) {
          verticalThrust -= 1;
        }
        
        if (verticalThrust !== 0) {
          const verticalPower = 800; // Moderate power for docked ship
          const liftForce = { x: 0, y: verticalThrust * verticalPower, z: 0 };
          stationShipBody.addForce(liftForce, true);
        }
        
        // Yaw rotation
        let yawTorque = 0;
        if (shipMovement.left) {
          yawTorque += 1;
        }
        if (shipMovement.right) {
          yawTorque -= 1;
        }
        
        if (yawTorque !== 0) {
          const torqueForce = { x: 0, y: yawTorque * torquePower, z: 0 };
          stationShipBody.addTorque(torqueForce, true);
        }
        
        // Check if ship has moved outside station boundaries - if so, automatically undock
        const currentStationShipPos = stationShipBody.translation();
        const stationLength = 250, stationWidth = 200, stationHeight = 60;
        const stationBounds = {
          minX: -stationWidth/2, maxX: stationWidth/2,
          minY: 1, maxY: stationHeight - 1,  
          minZ: -stationLength/2, maxZ: stationLength/2
        };
        
        const isOutsideStation = 
          currentStationShipPos.x < stationBounds.minX || currentStationShipPos.x > stationBounds.maxX ||
          currentStationShipPos.y < stationBounds.minY || currentStationShipPos.y > stationBounds.maxY ||
          currentStationShipPos.z < stationBounds.minZ || currentStationShipPos.z > stationBounds.maxZ;
        
        if (isOutsideStation) {
          console.log('SHIP OUTSIDE STATION BOUNDS - Auto-undocking:', currentStationShipPos);
          console.log('Station bounds:', stationBounds);
          undockShip();
          return; // Exit early since we just undocked
        }
      }
      return; // Skip exterior physics when docked
    }
    
    // Get ship's current rotation to apply thrust in local direction
    const shipRotation = shipBody.rotation();
    const shipQuat = new THREE.Quaternion(shipRotation.x, shipRotation.y, shipRotation.z, shipRotation.w);
    
    // Forward/backward thrust in ship's local forward direction
    let thrustZ = 0;
    if (shipMovement.forward) {
      thrustZ -= 1;
      console.log('Applying forward thrust');
    }
    if (shipMovement.backward) {
      thrustZ += 1;
      console.log('Applying backward thrust');
    }
    
    if (thrustZ !== 0) {
      const forwardVector = new THREE.Vector3(0, 0, thrustZ).applyQuaternion(shipQuat);
      
      // Try both force AND impulse for stronger effect
      const thrustForce = {
        x: forwardVector.x * thrustPower,
        y: forwardVector.y * thrustPower,
        z: forwardVector.z * thrustPower
      };
      const thrustImpulse = {
        x: forwardVector.x * thrustPower * deltaTime,
        y: forwardVector.y * thrustPower * deltaTime,
        z: forwardVector.z * thrustPower * deltaTime
      };
      
      console.log('Applying thrust - Force:', thrustForce, 'Impulse:', thrustImpulse);
      console.log('Ship position before thrust:', shipBody.translation());
      
      // Apply both force and impulse for maximum effect
      shipBody.addForce(thrustForce, true);
      shipBody.applyImpulse(thrustImpulse, true);
      
      console.log('Ship velocity after thrust:', shipBody.linvel());
    }
    
    // Vertical movement (Q/E for external flight, R/Y for interior piloting)
    let verticalThrust = 0;
    if (shipMovement.up) {
      verticalThrust += 1;
      // console.log('Applying upward thrust');
    }
    if (shipMovement.down) {
      verticalThrust -= 1;
      // console.log('Applying downward thrust');
    }
    
    if (verticalThrust !== 0) {
      // Use much more moderate vertical forces
      const verticalPower = isInsideShip ? 800 : liftPower; // Gentler when piloted from inside
      const liftForce = { x: 0, y: verticalThrust * verticalPower, z: 0 };
      const liftImpulse = { x: 0, y: verticalThrust * verticalPower * deltaTime, z: 0 };
      
      // console.log('Applying vertical thrust - Force:', liftForce, 'Impulse:', liftImpulse);
      
      // Apply both force and impulse
      shipBody.addForce(liftForce, true);
      shipBody.applyImpulse(liftImpulse, true);
    }
    
    // Yaw rotation (left/right arrows)
    let yawTorque = 0;
    if (shipMovement.left) {
      yawTorque += 1;
      console.log('Applying left yaw');
    }
    if (shipMovement.right) {
      yawTorque -= 1;
      console.log('Applying right yaw');
    }
    
    if (yawTorque !== 0) {
      const torqueForce = { x: 0, y: yawTorque * torquePower, z: 0 };
      const torqueImpulse = { x: 0, y: yawTorque * torquePower * deltaTime, z: 0 };
      
      console.log('Applying torque - Force:', torqueForce, 'Impulse:', torqueImpulse);
      console.log('Ship rotation before torque:', shipBody.rotation());
      
      // Apply both torque and torque impulse
      shipBody.addTorque(torqueForce, true);
      shipBody.applyTorqueImpulse(torqueImpulse, true);
      
      console.log('Ship angular velocity after torque:', shipBody.angvel());
    }
    
    // Update ship mesh position and rotation to match physics (only if not docked)
    if (!isShipDocked) {
      const pos = shipBody.translation();
      const rot = shipBody.rotation();
      shipMesh.position.set(pos.x, pos.y, pos.z);
      shipMesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);
    }
    // Note: When docked, ship mesh is updated in updateShip() from station physics
    
    // Interior proxy always stays at origin (static)
    if (isInsideShip) {
      shipInteriorProxy.position.set(0, 0, 0);
      interiorCamera.lookAt(0, 1.5, 0);
    }
    
    } catch (error) {
      console.warn('Error updating ship:', error);
    }
  }
  
  function updateStation(deltaTime) {
    const thrustPower = 3000; // Strong thrust power for large station
    const liftPower = 3500; // Strong lift power for large station  
    const torquePower = 1500; // Strong torque power for large station
    
    try {
    
    // Get station's current rotation to apply thrust in local direction
    const stationRotation = stationBody.rotation();
    const stationQuat = new THREE.Quaternion(stationRotation.x, stationRotation.y, stationRotation.z, stationRotation.w);
    
    // Forward/backward thrust in station's local forward direction
    let thrustZ = 0;
    if (stationMovement.forward) {
      thrustZ -= 1;
      console.log('Applying station forward thrust');
    }
    if (stationMovement.backward) {
      thrustZ += 1;
      console.log('Applying station backward thrust');
    }
    
    if (thrustZ !== 0) {
      const forwardVector = new THREE.Vector3(0, 0, thrustZ).applyQuaternion(stationQuat);
      
      const thrustForce = {
        x: forwardVector.x * thrustPower,
        y: forwardVector.y * thrustPower,
        z: forwardVector.z * thrustPower
      };
      const thrustImpulse = {
        x: forwardVector.x * thrustPower * deltaTime,
        y: forwardVector.y * thrustPower * deltaTime,
        z: forwardVector.z * thrustPower * deltaTime
      };
      
      console.log('Applying station thrust - Force:', thrustForce, 'Impulse:', thrustImpulse);
      
      stationBody.addForce(thrustForce, true);
      stationBody.applyImpulse(thrustImpulse, true);
    }
    
    // Yaw rotation (left/right)
    let yawTorque = 0;
    if (stationMovement.left) {
      yawTorque += 1;
      console.log('Applying station left yaw');
    }
    if (stationMovement.right) {
      yawTorque -= 1;
      console.log('Applying station right yaw');
    }
    
    if (yawTorque !== 0) {
      const torqueForce = { x: 0, y: yawTorque * torquePower, z: 0 };
      const torqueImpulse = { x: 0, y: yawTorque * torquePower * deltaTime, z: 0 };
      
      console.log('Applying station torque - Force:', torqueForce, 'Impulse:', torqueImpulse);
      
      stationBody.addTorque(torqueForce, true);
      stationBody.applyTorqueImpulse(torqueImpulse, true);
    }
    
    // Update station mesh position and rotation to match physics
    const pos = stationBody.translation();
    const rot = stationBody.rotation();
    stationMesh.position.set(pos.x, pos.y, pos.z);
    stationMesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);
    
    // Station interior proxy always stays at origin (static)
    stationInteriorProxy.position.set(0, 0, 0);
    
    // Update ship proxy position in station interior if ship is docked
    if (isShipDocked && stationShipProxy) {
      // Use station physics position for ship proxy (physics calculated in station world)
      const stationShipPos = stationShipBody.translation();
      const stationShipRot = stationShipBody.rotation();
      
      stationShipProxy.position.set(stationShipPos.x, stationShipPos.y, stationShipPos.z);
      stationShipProxy.quaternion.set(stationShipRot.x, stationShipRot.y, stationShipRot.z, stationShipRot.w);
      
      // Debug logging for alignment issues
      if (Math.random() < 0.01) { // 1% chance to avoid spam
        console.log('Station ship alignment check:');
        console.log('  Physics body pos:', { x: stationShipPos.x.toFixed(2), y: stationShipPos.y.toFixed(2), z: stationShipPos.z.toFixed(2) });
        console.log('  Visual proxy pos:', { x: stationShipProxy.position.x.toFixed(2), y: stationShipProxy.position.y.toFixed(2), z: stationShipProxy.position.z.toFixed(2) });
        console.log('  Proxy visible:', stationShipProxy.visible);
        console.log('  Ship floor should be at Y:', (stationShipPos.y - 0.4).toFixed(2), '(station floor at Y=1)');
        console.log('  Ship bounds: X±10, Y:', (stationShipPos.y - 1).toFixed(2), 'to', (stationShipPos.y + 6).toFixed(2), ', Z±15');
      }
      
      if (!stationShipProxy.visible) {
        console.log('WARNING: Station ship proxy not visible but should be!');
        console.log('Ship proxy position:', stationShipProxy.position);
        console.log('Station ship body position:', stationShipPos);
      }
    }
    
    } catch (error) {
      console.warn('Error updating station:', error);
    }
  }
  
  let lastTime = 0;
  
  function animate(currentTime = 0) {
    requestAnimationFrame(animate);
    
    const deltaTime = Math.min((currentTime - lastTime) / 1000, 1/30);
    lastTime = currentTime;
    
    // Update FPS
    frameCount++;
    if (currentTime - lastFpsTime >= 1000) {
      fps = Math.round(frameCount * 1000 / (currentTime - lastFpsTime));
      frameCount = 0;
      lastFpsTime = currentTime;
    }
    
    if (deltaTime > 0) {
      try {
        // Step exterior world with proper timestep
        exteriorWorld.timestep = deltaTime;
        exteriorWorld.step(exteriorEventQueue);
        
        // Process exterior collision events for ship entry detection and station interactions
        exteriorEventQueue.drainCollisionEvents((handle1, handle2, started) => {
          if (started) {
            try {
              const collider1 = exteriorWorld.getCollider(handle1);
              const collider2 = exteriorWorld.getCollider(handle2);
              
              // Safety check - ensure colliders are valid
              if (!collider1 || !collider2 || !playerCollider) return;
              
              // Check ship entry (player to ship door sensor) - only in exterior world
              if (!isInsideShip && !isInsideStation && !isExiting && !isTransitioning && !isEntering && (collider1 === playerCollider || collider2 === playerCollider)) {
                const otherCollider = collider1 === playerCollider ? collider2 : collider1;
                
                // Check if colliding with ship door sensor
                if (otherCollider && otherCollider.isSensor() && otherCollider.parent() === shipBody) {
                  const playerPos = playerBody.translation();
                  console.log('SHIP ENTRY DETECTED - Deferring entry until after physics step:', playerPos);
                  pendingShipEntry = true;
                }
                
                // Check if colliding with station entry sensor (with cooldown to prevent re-entry after ship exit) - only in exterior world
                if (otherCollider && otherCollider.isSensor() && otherCollider.parent() === stationBody && !isInsideStation) {
                  const currentTime = Date.now();
                  const timeSinceShipExit = currentTime - lastShipExitToStationTime;
                  
                  if (timeSinceShipExit > 1000) { // 1 second cooldown after ship exit to station
                    const playerPos = playerBody.translation();
                    console.log('STATION ENTRY DETECTED - Deferring entry until after physics step:', playerPos);
                    pendingStationEntry = true;
                  } else {
                    console.log('STATION ENTRY BLOCKED - Cooldown active after ship exit, time since exit:', timeSinceShipExit);
                  }
                }
              }
              
              // Check ship docking (ship to station docking sensor) 
              if (!isShipDocked && (collider1.parent() === shipBody || collider2.parent() === shipBody)) {
                const otherCollider = collider1.parent() === shipBody ? collider2 : collider1;
                
                // Debug: Log all ship collisions with station
                if (otherCollider && otherCollider.parent() === stationBody) {
                  const shipPos = shipBody.translation();
                  const stationPos = stationBody.translation();
                  console.log('Ship colliding with station component at ship pos:', shipPos, 'station pos:', stationPos);
                  console.log('Is sensor?', otherCollider.isSensor());
                }
                
                // Check if ship colliding with docking sensor (with cooldown to prevent immediate re-docking)
                if (otherCollider && otherCollider.isSensor() && otherCollider.parent() === stationBody) {
                  const currentTime = Date.now();
                  const timeSinceUndock = currentTime - lastUndockTime;
                  
                  if (timeSinceUndock > 2000) { // 2 second cooldown after undocking
                    const shipPos = shipBody.translation();
                    console.log('SHIP DOCKING DETECTED - Deferring docking until after physics step:', shipPos);
                    pendingShipDocking = true;
                  } else {
                    console.log('SHIP DOCKING BLOCKED - Cooldown active, time since undock:', timeSinceUndock);
                  }
                }
              }
            } catch (error) {
              console.warn('Error processing exterior collision events:', error);
            }
          }
        });
        
        // Step interior world with proper timestep
        interiorWorld.timestep = deltaTime;
        interiorWorld.step(interiorEventQueue);
        
        // Step station world with proper timestep
        stationWorld.timestep = deltaTime;
        stationWorld.step(stationEventQueue);
        
        // Check for automatic exit based on momentum and clear path
        if (isInsideShip && !isTransitioning) {
          checkAutoExit();
        }
        
        if (isInsideStation && !isTransitioning) {
          checkStationAutoExit();
        }
        
        // Process interior collision events for sensor detection
        interiorEventQueue.drainCollisionEvents(() => {
          // Remove sensor-based exit detection - now using momentum + raycast system
        });
        
        // Process station collision events  
        stationEventQueue.drainCollisionEvents((collider1, collider2, started) => {
          if (!started) return; // Only process collision start events
          
          try {
            // Safety check - ensure colliders are valid
            if (!collider1 || !collider2) return;
            
            // Debug: Log collisions in station world (when ship entry might be affected)
            if (isInsideStation && isShipDocked && Math.random() < 0.1) { // 10% chance when conditions are right
              const isPlayerCollision = stationPlayerCollider && (collider1 === stationPlayerCollider || collider2 === stationPlayerCollider);
              if (isPlayerCollision) {
                const otherCollider = collider1 === stationPlayerCollider ? collider2 : collider1;
                console.log('🌟 Station player collision detected:');  
                console.log('  Other collider parent:', otherCollider?.parent() === stationShipBody ? 'Ship' : 
                           otherCollider?.parent() === stationBody ? 'Station' : 'Other');
                console.log('  Player conditions for ship entry:', { 
                  isInsideStation, 
                  isShipDocked, 
                  isInsideShip, 
                  isExiting, 
                  isTransitioning, 
                  isEntering, 
                  pendingShipEntryFromStation 
                });
                
                // Check if this collision might be interfering with ship entry
                if (otherCollider?.parent() === stationBody) {
                  console.log('  → Player colliding with station walls/floor - this is normal');
                } else if (otherCollider?.parent() === stationShipBody) {
                  console.log('  → Player colliding with ship in station - checking if this affects entry logic');
                }
              }
            }
            
            // Position-based ship entry detection moved to main frame loop
            // No collision-based detection needed here
          } catch (error) {
            console.warn('Error processing station collision events:', error);
          }
        });
        
        // Update transition progress
        if (isTransitioning) {
          updateTransition(deltaTime);
        }
        
        // Ship docking is now immediate - no transition updates needed
        
        // Process deferred entry actions after all physics steps are complete
        if (pendingShipEntry && !isEntering) {
          pendingShipEntry = false;
          console.log('PROCESSING DEFERRED SHIP ENTRY');
          immediateEnterInterior();
        }
        
        if (pendingStationEntry && !isEntering) {
          pendingStationEntry = false;
          console.log('PROCESSING DEFERRED STATION ENTRY');
          immediateEnterStation();
        }
        
        if (pendingShipDocking && !isShipDocked) {
          pendingShipDocking = false;
          console.log('PROCESSING DEFERRED SHIP DOCKING');
          startShipDockingTransition();
        }
        
        // Position-based ship entry detection (runs every frame) - only when ship is docked in station
        if (isInsideStation && isShipDocked && !isInsideShip && !isExiting && !isTransitioning && !isEntering && !pendingShipEntryFromStation) {
          const playerPos = stationPlayerBody.translation();
          const shipPos = stationShipBody.translation();
          
          // Check if player is within ship interior bounds (station world coordinates)
          const relativeToShip = {
            x: playerPos.x - shipPos.x,
            y: playerPos.y - shipPos.y,
            z: playerPos.z - shipPos.z
          };
          
          // Ship interior bounds: width=20, height=6, length=30
          const shipWidth = 20, shipHeight = 6, shipLength = 30;
          const xInBounds = Math.abs(relativeToShip.x) < shipWidth/2;
          const yInBounds = relativeToShip.y > -1 && relativeToShip.y < shipHeight;
          const zInBounds = Math.abs(relativeToShip.z) < shipLength/2;
          const isPlayerInsideShipBounds = xInBounds && yInBounds && zInBounds;
          
          // Debug logging every few frames to see what's happening
          if (frameCount % 60 === 0) { // Every 1 second at 60fps
            console.log('🔍 Position-based ship entry check:');
            console.log('  Player pos in station:', {x: playerPos.x.toFixed(2), y: playerPos.y.toFixed(2), z: playerPos.z.toFixed(2)});
            console.log('  Ship pos in station:', {x: shipPos.x.toFixed(2), y: shipPos.y.toFixed(2), z: shipPos.z.toFixed(2)});
            console.log('  Player relative to ship:', {x: relativeToShip.x.toFixed(2), y: relativeToShip.y.toFixed(2), z: relativeToShip.z.toFixed(2)});
            console.log('  Bounds check (width=20, height=6, length=30):', {
              xOK: xInBounds, xValue: Math.abs(relativeToShip.x).toFixed(2), xLimit: (shipWidth/2).toFixed(2),
              yOK: yInBounds, yValue: relativeToShip.y.toFixed(2), yRange: '-1 to 6',
              zOK: zInBounds, zValue: Math.abs(relativeToShip.z).toFixed(2), zLimit: (shipLength/2).toFixed(2)
            });
            console.log('  Overall inside bounds:', isPlayerInsideShipBounds);
            console.log('  Conditions check:', { 
              isInsideStation, 
              isShipDocked, 
              isInsideShip, 
              isExiting, 
              isTransitioning, 
              isEntering, 
              pendingShipEntryFromStation 
            });
            
            // Check if the main condition is blocking
            const mainCondition = isInsideStation && isShipDocked && !isInsideShip && !isExiting && !isTransitioning && !isEntering && !pendingShipEntryFromStation;
            console.log('  Main condition satisfied:', mainCondition);
            if (!mainCondition) {
              console.log('  → Blocked by:', 
                !isInsideStation ? 'not in station' :
                !isShipDocked ? 'ship not docked' :
                isInsideShip ? 'already in ship' :
                isExiting ? 'currently exiting' :
                isTransitioning ? 'currently transitioning' :
                isEntering ? 'currently entering' :
                pendingShipEntryFromStation ? 'entry already pending' :
                'unknown reason');
            }
          }
          
          if (isPlayerInsideShipBounds) {
            console.log('🎯 SHIP ENTRY FROM STATION DETECTED (position-based):');
            console.log('  Triggering ship entry transition...');
            pendingShipEntryFromStation = true;
          }
        }
        
        if (pendingShipEntryFromStation && !isEntering) {
          pendingShipEntryFromStation = false;
          console.log('🎯 PROCESSING DEFERRED SHIP ENTRY FROM STATION - Calling enterShipFromStation()');
          enterShipFromStation();
        } else if (pendingShipEntryFromStation && isEntering) {
          console.log('⏳ Ship entry from station pending but isEntering=true, waiting...');
        }
        
        // Debug player state every few seconds
        if (frameCount % 300 === 0) { // Every ~5 seconds at 60fps
          console.log('🎮 PLAYER STATE DEBUG:', {
            isInsideShip,
            isInsideStation, 
            isEntering,
            isExiting,
            isTransitioning,
            frameCount
          });
          
          // Debug station ship entry position
          if (isInsideStation && isShipDocked) {
            const playerPos = stationPlayerBody.translation();
            const shipPos = stationShipBody.translation();
            console.log('🚪 STATION SHIP ENTRY DEBUG:');
            console.log('  Player position:', {x: playerPos.x.toFixed(2), y: playerPos.y.toFixed(2), z: playerPos.z.toFixed(2)});
            console.log('  Ship position:', {x: shipPos.x.toFixed(2), y: shipPos.y.toFixed(2), z: shipPos.z.toFixed(2)});
            
            // Calculate relative position for bounds checking
            const relativeToShip = {
              x: playerPos.x - shipPos.x,
              y: playerPos.y - shipPos.y,
              z: playerPos.z - shipPos.z
            };
            console.log('  Player relative to ship:', {x: relativeToShip.x.toFixed(2), y: relativeToShip.y.toFixed(2), z: relativeToShip.z.toFixed(2)});
            console.log('  Within ship bounds?', {
              xOK: Math.abs(relativeToShip.x) < 10,
              yOK: relativeToShip.y > -1 && relativeToShip.y < 6,
              zOK: Math.abs(relativeToShip.z) < 15
            });
          }
        }
        
        updatePlayer(deltaTime);
        updateShip(deltaTime);
        updateStation(deltaTime);
        
      } catch (error) {
        console.error('Critical physics error:', error);
        // Skip this frame to prevent cascade failures
        return;
      }
    }
    
    // Always render triple view - all world cameras always visible
    renderTripleView();
  }
  
  function renderShipInteriorView() {
    const width = window.innerWidth;
    const height = window.innerHeight;
    
    // Update camera aspect ratio for full screen
    interiorCamera.aspect = width / height;
    interiorCamera.updateProjectionMatrix();
    
    // Full screen ship interior view
    renderer.setViewport(0, 0, width, height);
    renderer.setScissorTest(false);
    renderer.clear();
    
    // Show only ship interior proxy
    shipMesh.visible = false;
    stationMesh.visible = false;
    shipInteriorProxy.visible = true;
    stationInteriorProxy.visible = false;
    
    // Show debug player mesh based on interior position
    const interiorPos = interiorPlayerBody.translation();
    if (interiorPos.y > -50) {
      debugPlayerMesh.position.set(interiorPos.x, interiorPos.y, interiorPos.z);
      debugPlayerMesh.visible = true;
    } else {
      debugPlayerMesh.visible = false;
    }
    
    renderer.render(interiorScene, interiorCamera);
  }
  
  function renderStationInteriorView() {
    const width = window.innerWidth;
    const height = window.innerHeight;
    
    // Update camera aspect ratio for full screen
    stationCamera.aspect = width / height;
    stationCamera.updateProjectionMatrix();
    
    // Full screen station interior view
    renderer.setViewport(0, 0, width, height);
    renderer.setScissorTest(false);
    renderer.clear();
    
    // Show only station interior proxy
    shipMesh.visible = false;
    stationMesh.visible = false;
    shipInteriorProxy.visible = false;
    stationInteriorProxy.visible = true;
    
    // Show station ship proxy if docked
    if (stationShipProxy) {
      stationShipProxy.visible = isShipDocked;
    }
    
    // Show debug station player mesh based on station position
    const stationPos = stationPlayerBody.translation();
    if (stationPos.y > -50) {
      debugStationPlayerMesh.position.set(stationPos.x, stationPos.y, stationPos.z);
      debugStationPlayerMesh.visible = true;
    } else {
      debugStationPlayerMesh.visible = false;
    }
    
    renderer.render(stationScene, stationCamera);
  }

  function renderTripleView() {
    const width = window.innerWidth;
    const height = window.innerHeight;
    
    // Update camera aspect ratios for 1/3 width viewports
    const viewportAspect = (width / 3) / height;
    exteriorCamera.aspect = viewportAspect;
    exteriorCamera.updateProjectionMatrix();
    interiorCamera.aspect = viewportAspect;
    interiorCamera.updateProjectionMatrix();
    stationCamera.aspect = viewportAspect;
    stationCamera.updateProjectionMatrix();
    
    // Clear the entire screen
    renderer.setScissorTest(false);
    renderer.clear();
    
    // Left viewport - Exterior world
    renderer.setViewport(0, 0, width / 3, height);
    renderer.setScissor(0, 0, width / 3, height);
    renderer.setScissorTest(true);
    
    // Show exterior ship and station, hide interior proxies
    shipMesh.visible = true;
    stationMesh.visible = true;
    shipInteriorProxy.visible = false;
    stationInteriorProxy.visible = false;
    // Always show player mesh in exterior world (positioned correctly whether inside or outside ship/station)
    playerMesh.visible = true;
    
    renderer.render(exteriorScene, exteriorCamera);
    
    // Middle viewport - Ship Interior world view
    renderer.setViewport(width / 3, 0, width / 3, height);
    renderer.setScissor(width / 3, 0, width / 3, height);
    
    // Show ship interior proxy and hide exterior meshes
    shipMesh.visible = false;
    stationMesh.visible = false;
    shipInteriorProxy.visible = true;
    stationInteriorProxy.visible = false;
    
    // Show debug player mesh based on transition progress and if positioned correctly
    const interiorPos = interiorPlayerBody.translation();
    const shouldShowDebugPlayer = (isInsideShip || (isTransitioning && transitionProgress > 0.3)) && interiorPos.y > -50;
    
    if (shouldShowDebugPlayer) {
      debugPlayerMesh.position.set(interiorPos.x, interiorPos.y, interiorPos.z);
      debugPlayerMesh.visible = true;
    } else {
      debugPlayerMesh.visible = false;
    }
    
    renderer.render(interiorScene, interiorCamera);
    
    // Right viewport - Station Interior world view
    renderer.setViewport(2 * width / 3, 0, width / 3, height);
    renderer.setScissor(2 * width / 3, 0, width / 3, height);
    
    // Show station interior proxy and hide exterior meshes
    shipMesh.visible = false;
    stationMesh.visible = false;
    shipInteriorProxy.visible = false;
    stationInteriorProxy.visible = true;
    
    // Show debug station player mesh if in station
    const stationPos = stationPlayerBody.translation();
    const shouldShowStationPlayer = isInsideStation && stationPos.y > -150;
    
    if (shouldShowStationPlayer) {
      debugStationPlayerMesh.position.set(stationPos.x, stationPos.y, stationPos.z);
      debugStationPlayerMesh.visible = true;
    } else {
      debugStationPlayerMesh.visible = false;
    }
    
    // Ensure ship proxy visibility is correct based on docking state
    if (stationShipProxy) {
      stationShipProxy.visible = isShipDocked;
    }
    
    renderer.render(stationScene, stationCamera);
    
    // Reset viewport and scissor
    renderer.setViewport(0, 0, width, height);
    renderer.setScissorTest(false);
    
    // Restore visibility for next frame
    playerMesh.visible = true;
    shipMesh.visible = true;
    stationMesh.visible = true;
  }
  
  function onWindowResize() {
    // Update aspect ratio for triple screen (third width)
    exteriorCamera.aspect = (window.innerWidth / 3) / window.innerHeight;
    exteriorCamera.updateProjectionMatrix();
    interiorCamera.aspect = (window.innerWidth / 3) / window.innerHeight;
    interiorCamera.updateProjectionMatrix();
    stationCamera.aspect = (window.innerWidth / 3) / window.innerHeight;
    stationCamera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }
</script>

<canvas bind:this={canvas}></canvas>

<div class="ui">
  <p>WASD: Move | Space: Jump | Arrow Keys: Fly Ship | Q/E: Up/Down | O: Toggle Camera | Click: Pointer Lock | ESC: Exit Lock</p>
  <p>TFGH + RY: Ship Movement (when inside ship) | IKJL: Station Movement (when inside station)</p>
  <p>Status: {isInsideShip ? (isShipDocked ? 'Inside Ship (Docked in Station)' : 'Inside Ship (Free Flight)') : isInsideStation ? 'Inside Station' : 'Outside/Exterior World'}</p>
  <p>Ship Status: {isShipDocked ? 'Docked in Station' : 'Free Flight'}</p>
  <p>Transitions: {isTransitioning ? 'Player Transitioning' : ''} {isEntering ? 'Entering' : ''} {isExiting ? 'Exiting' : ''}</p>
  <p>Movement: {movementStatus}</p>
  <p>Camera: {isFirstPerson ? 'First Person' : 'Third Person'} | Pointer Lock: {isPointerLocked ? 'ON' : 'OFF'}</p>
</div>

<div class="camera-labels">
  <div class="camera-label left">Exterior World</div>
  <div class="camera-label center">Ship Interior</div>
  <div class="camera-label right">Station Interior</div>
</div>

<div class="debug-hud">
  <h3>Debug Info</h3>
  <div class="debug-row">
    <span class="label">FPS:</span>
    <span class="value">{fps}</span>
  </div>
  <div class="debug-row">
    <span class="label">World:</span>
    <span class="value">{currentWorld}</span>
  </div>
  <div class="debug-row">
    <span class="label">Position:</span>
    <span class="value">({playerPosition.x}, {playerPosition.y}, {playerPosition.z})</span>
  </div>
  <div class="debug-row">
    <span class="label">Velocity:</span>
    <span class="value">({playerVelocity.x}, {playerVelocity.y}, {playerVelocity.z})</span>
  </div>
  <div class="debug-row">
    <span class="label">Movement:</span>
    <span class="value">{movementStatus}</span>
  </div>
  <div class="debug-row">
    <span class="label">Ship World:</span>
    <span class="value">{shipWorld}</span>
  </div>
  <div class="debug-row">
    <span class="label">Can Jump:</span>
    <span class="value">{canJump ? 'Yes' : 'No'}</span>
  </div>
  <div class="debug-row">
    <span class="label">Jump Pressed:</span>
    <span class="value">{jumpPressed ? 'Yes' : 'No'}</span>
  </div>
  <div class="debug-row">
    <span class="label">Inside Ship:</span>
    <span class="value">{isInsideShip ? 'YES' : 'NO'}</span>
  </div>
</div>

<!-- Crosshair for FPS mode -->
{#if isFirstPerson && isPointerLocked}
<div class="crosshair">
  <div class="crosshair-dot"></div>
  <div class="crosshair-lines">
    <div class="crosshair-line horizontal"></div>
    <div class="crosshair-line vertical"></div>
  </div>
</div>
{/if}

<style>
  canvas {
    display: block;
  }
  
  .ui {
    position: absolute;
    top: 10px;
    left: 10px;
    color: white;
    font-family: Arial, sans-serif;
    text-shadow: 2px 2px 4px rgba(0,0,0,0.8);
    z-index: 1000;
  }
  
  .ui p {
    margin: 5px 0;
  }
  
  .debug-hud {
    position: absolute;
    top: 10px;
    right: 10px;
    background: rgba(0, 0, 0, 0.8);
    color: white;
    padding: 15px;
    border-radius: 8px;
    font-family: 'Courier New', monospace;
    font-size: 12px;
    min-width: 250px;
    z-index: 1000;
  }
  
  .debug-hud h3 {
    margin: 0 0 10px 0;
    color: #00ff00;
    font-size: 14px;
    text-align: center;
  }
  
  .debug-row {
    display: flex;
    justify-content: space-between;
    margin: 4px 0;
    padding: 2px 0;
    border-bottom: 1px solid #333;
  }
  
  .debug-row:last-child {
    border-bottom: none;
  }
  
  .label {
    color: #888;
    font-weight: bold;
  }
  
  .value {
    color: #fff;
    text-align: right;
  }
  
  .camera-labels {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 40px;
    pointer-events: none;
    z-index: 100;
  }
  
  .camera-label {
    position: absolute;
    top: 10px;
    background: rgba(0, 0, 0, 0.7);
    color: white;
    padding: 5px 15px;
    border-radius: 15px;
    font-family: Arial, sans-serif;
    font-size: 14px;
    font-weight: bold;
  }
  
  .camera-label.left {
    left: 20px;
    color: #ffaa00;
  }
  
  .camera-label.center {
    left: 50%;
    transform: translateX(-50%);
    color: #00ff00;
  }
  
  .camera-label.right {
    right: 20px;
    color: #00aaff;
  }
  
  .crosshair {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    z-index: 1000;
    pointer-events: none;
  }
  
  .crosshair-dot {
    width: 4px;
    height: 4px;
    background: rgba(255, 255, 255, 0.8);
    border-radius: 50%;
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    box-shadow: 0 0 2px rgba(0, 0, 0, 0.8);
  }
  
  .crosshair-lines {
    position: relative;
  }
  
  .crosshair-line {
    background: rgba(255, 255, 255, 0.6);
    position: absolute;
    box-shadow: 0 0 1px rgba(0, 0, 0, 0.8);
  }
  
  .crosshair-line.horizontal {
    width: 20px;
    height: 1px;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
  }
  
  .crosshair-line.vertical {
    width: 1px;
    height: 20px;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
  }
</style>