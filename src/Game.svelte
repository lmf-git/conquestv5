<script>
  import { onMount } from 'svelte';
  import * as THREE from 'three';
  import RAPIER from '@dimforge/rapier3d-compat';
  
  let canvas = $state();
  let exteriorScene, interiorScene, exteriorCamera, interiorCamera, renderer;
  let exteriorWorld, interiorWorld, exteriorEventQueue, interiorEventQueue;
  let playerBody, playerCollider;
  let interiorPlayerBody, interiorPlayerCollider;
  let shipBody, shipCollider;
  let isInsideShip = $state(false);
  let shipInteriorProxy;
  let transitionProgress = $state(0); // 0 = fully outside, 1 = fully inside
  let isTransitioning = $state(false);
  let isExiting = $state(false);
  
  onMount(async () => {
    await RAPIER.init();
    
    // Create separate scenes for exterior and interior
    exteriorScene = new THREE.Scene();
    exteriorScene.background = new THREE.Color(0x87CEEB);
    exteriorScene.fog = new THREE.Fog(0x87CEEB, 10, 1000);
    
    interiorScene = new THREE.Scene();
    interiorScene.background = new THREE.Color(0x222222); // Dark interior background
    
    exteriorCamera = new THREE.PerspectiveCamera(75, (window.innerWidth / 2) / window.innerHeight, 0.1, 1000);
    exteriorCamera.position.set(0, 5, 10);
    
    interiorCamera = new THREE.PerspectiveCamera(75, (window.innerWidth / 2) / window.innerHeight, 0.1, 1000);
    interiorCamera.position.set(0, 4, 12);
    
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
    
    const gravity = { x: 0.0, y: -9.81, z: 0.0 };
    exteriorWorld = new RAPIER.World(gravity);
    exteriorEventQueue = new RAPIER.EventQueue(true);
    
    interiorWorld = new RAPIER.World(gravity);
    interiorEventQueue = new RAPIER.EventQueue(true);
    
    createGround();
    createPlayer();
    createShip();
    
    window.addEventListener('resize', onWindowResize);
    document.addEventListener('keydown', handleKeyDown);
    document.addEventListener('keyup', handleKeyUp);
    
    animate();
    
    return () => {
      window.removeEventListener('resize', onWindowResize);
      document.removeEventListener('keydown', handleKeyDown);
      document.removeEventListener('keyup', handleKeyUp);
    };
  });
  
  function createGround() {
    const groundGeometry = new THREE.BoxGeometry(100, 1, 100);
    const groundMaterial = new THREE.MeshStandardMaterial({ color: 0x808080 });
    const groundMesh = new THREE.Mesh(groundGeometry, groundMaterial);
    groundMesh.receiveShadow = true;
    groundMesh.position.y = -0.5;
    exteriorScene.add(groundMesh);
    
    const groundBodyDesc = RAPIER.RigidBodyDesc.fixed()
      .setTranslation(0, -0.5, 0);
    const groundBody = exteriorWorld.createRigidBody(groundBodyDesc);
    const groundColliderDesc = RAPIER.ColliderDesc.cuboid(50, 0.5, 50);
    exteriorWorld.createCollider(groundColliderDesc, groundBody);
  }
  
  let playerMesh;
  let movement = { forward: false, backward: false, left: false, right: false, jump: false };
  let shipMovement = { forward: false, backward: false, left: false, right: false, up: false, down: false };
  let isGrounded = $state(false);
  let jumpPressed = false;
  let canJump = true;
  
  // Debug HUD reactive states
  let playerPosition = $state({ x: 0, y: 0, z: 0 });
  let playerVelocity = $state({ x: 0, y: 0, z: 0 });
  let currentWorld = $derived(isInsideShip ? 'Interior' : 'Exterior');
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
    const exteriorDoorSensorCollider = RAPIER.ColliderDesc.cuboid(doorWidth / 2, doorHeight / 2, 1)
      .setTranslation(0, doorHeight / 2, shipLength / 2 + 1)
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
    
    // Optional: Add subtle debug markers to show physics floor bounds (very transparent)
    const debugFloorGeometry = new THREE.BoxGeometry(
      shipWidth - wallThickness * 2, 
      0.02, 
      shipLength - wallThickness * 2
    );
    const debugFloorMaterial = new THREE.MeshStandardMaterial({ 
      color: 0xff0000, 
      transparent: true, 
      opacity: 0.1 // Very faint
    });
    const debugFloorMesh = new THREE.Mesh(debugFloorGeometry, debugFloorMaterial);
    debugFloorMesh.position.y = wallThickness / 2 + 0.01; // Slightly above physics floor
    shipInteriorProxy.add(debugFloorMesh);
    
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
        console.log('Arrow up pressed');
        break;
      case 'arrowdown': 
        shipMovement.backward = true; 
        console.log('Arrow down pressed');
        break;
      case 'arrowleft': 
        shipMovement.left = true; 
        console.log('Arrow left pressed');
        break;
      case 'arrowright': 
        shipMovement.right = true; 
        console.log('Arrow right pressed');
        break;
      case 'q': 
        shipMovement.up = true; 
        console.log('Q pressed');
        break;
      case 'e': 
        shipMovement.down = true; 
        console.log('E pressed');
        break;
    }
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
    }
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
    exteriorWorld.removeRigidBody(playerBody);
    const kinematicPlayerBodyDesc = RAPIER.RigidBodyDesc.kinematicPositionBased()
      .setTranslation(currentExteriorPos.x, currentExteriorPos.y, currentExteriorPos.z);
    playerBody = exteriorWorld.createRigidBody(kinematicPlayerBodyDesc);
    
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const kinematicPlayerColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius);
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

  function checkAutoExit() {
    // Prevent multiple exit calls
    if (isExiting || !isInsideShip) return;
    
    const interiorPos = interiorPlayerBody.translation();
    const interiorVel = interiorPlayerBody.linvel();
    const shipLength = 30;
    const doorZ = shipLength / 2 - 0.4; // Front door position
    
    // Natural exit detection - when actually passing through door opening
    const inDoorOpening = Math.abs(interiorPos.x) < 2 && interiorPos.z > doorZ - 1; // Actually in door opening
    const movingTowardExit = interiorVel.z > 0.5; // Moving with purpose toward exit
    
    if (inDoorOpening && movingTowardExit) {
      console.log('INSTANT EXIT TRIGGERED - Player near door with exit momentum:', { interiorPos, interiorVel });
      
      // Lock exit to prevent multiple calls
      isExiting = true;
      
      // NO RAYCAST DELAY - exit immediately when conditions met
      console.log('IMMEDIATE EXIT - NO DELAYS');
      immediateExitToExterior();
    }
  }
  
  function immediateExitToExterior() {
    console.log('Immediate exit to exterior - creating dynamic body FIRST');
    
    // Get current interior state
    const currentInteriorPos = interiorPlayerBody.translation();
    const currentInteriorVel = interiorPlayerBody.linvel();
    const shipPos = shipBody.translation();
    const shipRot = shipBody.rotation();
    const shipQuat = new THREE.Quaternion(shipRot.x, shipRot.y, shipRot.z, shipRot.w);
    
    // Use player's actual interior position transformed to world coordinates (no teleportation)
    const relativePos = new THREE.Vector3(currentInteriorPos.x, currentInteriorPos.y, currentInteriorPos.z);
    relativePos.applyQuaternion(shipQuat);
    
    const exteriorX = shipPos.x + relativePos.x;
    const exteriorY = Math.max(shipPos.y + relativePos.y, 1); // Just ensure above ground level
    const exteriorZ = shipPos.z + relativePos.z;
    
    console.log('Creating dynamic body at safe position:', { exteriorX, exteriorY, exteriorZ });
    
    // CREATE DYNAMIC BODY FIRST - before removing kinematic
    const newDynamicBodyDesc = RAPIER.RigidBodyDesc.dynamic()
      .setTranslation(exteriorX, exteriorY, exteriorZ)
      .setCanSleep(false)
      .setLinearDamping(2.0)
      .setAngularDamping(5.0)
      .lockRotations();
    const newDynamicBody = exteriorWorld.createRigidBody(newDynamicBodyDesc);
    
    // Create collider for new dynamic body immediately with same collision groups as original
    const capsuleRadius = 0.5;
    const capsuleHeight = 1.5;
    const newDynamicColliderDesc = RAPIER.ColliderDesc.capsule(capsuleHeight / 2, capsuleRadius)
      .setActiveEvents(RAPIER.ActiveEvents.COLLISION_EVENTS)
      .setFriction(0.8)
      .setRestitution(0.1);
    const newDynamicCollider = exteriorWorld.createCollider(newDynamicColliderDesc, newDynamicBody);
    
    console.log('Dynamic body and collider created, forcing physics update...');
    
    // FORCE IMMEDIATE PHYSICS PROCESSING
    exteriorWorld.timestep = 0.016;
    exteriorWorld.step(exteriorEventQueue);
    
    console.log('Physics step completed, body position:', newDynamicBody.translation());
    
    // Apply exit velocity to new body AFTER physics step
    const exitVel = new THREE.Vector3(currentInteriorVel.x, currentInteriorVel.y, currentInteriorVel.z);
    exitVel.applyQuaternion(shipQuat);
    newDynamicBody.setLinvel({ x: exitVel.x, y: exitVel.y, z: exitVel.z }, true);
    
    // Force another physics step to ensure velocity is applied and collisions are detected
    exteriorWorld.timestep = 0.016;
    exteriorWorld.step(exteriorEventQueue);
    
    console.log('After velocity application - position:', newDynamicBody.translation());
    console.log('After velocity application - velocity:', newDynamicBody.linvel());
    
    console.log('New dynamic body created with handle:', newDynamicBody.handle);
    console.log('New collider created with handle:', newDynamicCollider.handle);
    
    // NOW remove old kinematic body (after new one is established)
    console.log('Removing old kinematic body...');
    if (playerCollider) {
      // Change old collider to different group to prevent conflicts
      exteriorWorld.removeCollider(playerCollider, false);
    }
    exteriorWorld.removeRigidBody(playerBody);
    
    // Switch references to new dynamic body IMMEDIATELY
    playerBody = newDynamicBody;
    playerCollider = newDynamicCollider;
    isInsideShip = false;
    isExiting = false; // Reset exit lock
    
    // Apply small upward impulse to test collision detection
    playerBody.applyImpulse({ x: 0, y: 2, z: 0 }, true);
    
    // Force one more physics step to ensure collision is registered
    exteriorWorld.timestep = 0.016;
    exteriorWorld.step(exteriorEventQueue);
    
    console.log('Switched to dynamic body - type:', playerBody.bodyType());
    console.log('Position after impulse test:', playerBody.translation());
    console.log('Velocity after impulse test:', playerBody.linvel());
    
    // Immediate ground check
    const rayStart = { x: exteriorX, y: exteriorY, z: exteriorZ };
    const rayDir = { x: 0, y: -1, z: 0 };
    const ray = new RAPIER.Ray(rayStart, rayDir);
    const groundHit = exteriorWorld.castRay(ray, 10, true);
    
    if (groundHit) {
      console.log('Ground confirmed at distance:', groundHit.toi);
    } else {
      console.log('WARNING: No ground detected - adjusting position');
      // If no ground, spawn at known safe position
      playerBody.setTranslation({ x: 0, y: 5, z: 25 }, true);
    }
  }

  function exitShip() {
    console.log('Seamlessly exiting ship - switching to exterior physics control');
    isInsideShip = false;
    
    // Get current interior player state
    const currentInteriorPos = interiorPlayerBody.translation();
    const currentInteriorVel = interiorPlayerBody.linvel();
    const shipPos = shipBody.translation();
    
    // Calculate exterior world position with ship rotation
    const shipRot = shipBody.rotation();
    const shipQuat = new THREE.Quaternion(shipRot.x, shipRot.y, shipRot.z, shipRot.w);
    
    // Apply ship rotation to interior relative position
    const relativePos = new THREE.Vector3(currentInteriorPos.x, currentInteriorPos.y, currentInteriorPos.z);
    relativePos.applyQuaternion(shipQuat);
    
    // Exit player at a safe position outside the ship (front door area)
    const safeExitOffset = new THREE.Vector3(0, 1, 18); // In front of ship, above ground
    safeExitOffset.applyQuaternion(shipQuat);
    
    const exteriorX = shipPos.x + safeExitOffset.x;
    const exteriorY = Math.max(shipPos.y + safeExitOffset.y, 2); // Ensure above ground
    const exteriorZ = shipPos.z + safeExitOffset.z;
    
    console.log('Switching control - exterior position:', { exteriorX, exteriorY, exteriorZ });
    
    // Convert kinematic player back to dynamic body at current position
    console.log('Removing kinematic body...');
    exteriorWorld.removeRigidBody(playerBody);
    
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
  
  function checkGrounded() {
    if (isInsideShip) {
      const playerPos = interiorPlayerBody.translation();
      const rayStart = { x: playerPos.x, y: playerPos.y - 0.65, z: playerPos.z };
      const rayDir = { x: 0, y: -1, z: 0 };
      const rayLength = 0.3;
      
      const ray = new RAPIER.Ray(rayStart, rayDir);
      const hit = interiorWorld.castRay(ray, rayLength, true);
      
      isGrounded = hit !== null;
      
      // Debug interior physics (only every 60 frames to avoid spam)
      if (Math.random() < 0.016) { // ~1/60 chance
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
    } else {
      const playerPos = playerBody.translation();
      const rayStart = { x: playerPos.x, y: playerPos.y - 0.65, z: playerPos.z };
      const rayDir = { x: 0, y: -1, z: 0 };
      const rayLength = 0.3;
      
      const ray = new RAPIER.Ray(rayStart, rayDir);
      const hit = exteriorWorld.castRay(ray, rayLength, true);
      
      isGrounded = hit !== null;
    }
  }
  
  function updatePlayer(deltaTime) {
    const groundSpeed = 4;
    const airSpeed = 1.5; // Slower movement in air
    const jumpSpeed = 7;
    
    checkGrounded();
    
    let velocity, currentPlayerBody;
    
    if (isInsideShip) {
      currentPlayerBody = interiorPlayerBody;
      velocity = interiorPlayerBody.linvel();
    } else {
      currentPlayerBody = playerBody;
      velocity = playerBody.linvel();
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
      
      // Position player mesh in real world interior (translated from static proxy)
      playerMesh.position.set(exteriorPos.x, exteriorPos.y, exteriorPos.z);
      
      // Interior camera stays fixed
      interiorCamera.position.set(0, 4, 12);
      interiorCamera.lookAt(0, 1.5, 0);
      
      // Exterior camera follows real world player representation when inside ship
      exteriorCamera.position.set(exteriorPos.x, exteriorPos.y + 5, exteriorPos.z + 10);
      exteriorCamera.lookAt(exteriorPos.x, exteriorPos.y + 1, exteriorPos.z);
    } else {
      // Position player mesh for exterior view
      playerMesh.position.set(pos.x, pos.y, pos.z);
      
      // Hide debug player mesh when outside
      debugPlayerMesh.visible = false;
      
      // Update cameras - exterior camera follows player position
      // During exit transition, make sure we use the correct player body position
      const cameraTargetPos = isTransitioning ? playerBody.translation() : pos;
      exteriorCamera.position.set(cameraTargetPos.x, cameraTargetPos.y + 5, cameraTargetPos.z + 10);
      exteriorCamera.lookAt(cameraTargetPos.x, cameraTargetPos.y + 1, cameraTargetPos.z);
      
      if (isTransitioning) {
        console.log('Transitioning - exterior camera following playerBody at:', cameraTargetPos);
      }
    }
    
    // Update debug states
    playerPosition = { x: Math.round(pos.x * 100) / 100, y: Math.round(pos.y * 100) / 100, z: Math.round(pos.z * 100) / 100 };
    playerVelocity = { x: Math.round(velocity.x * 100) / 100, y: Math.round(velocity.y * 100) / 100, z: Math.round(velocity.z * 100) / 100 };
  }
  
  let shipTime = 0;
  
  function updateShip(deltaTime) {
    const thrustPower = 150;
    const liftPower = 200; // Counter-gravity force
    const torquePower = 50;
    
    console.log('Ship movement state:', shipMovement);
    
    // Get ship's current rotation to apply thrust in local direction
    const shipRotation = shipBody.rotation();
    const shipQuat = new THREE.Quaternion(shipRotation.x, shipRotation.y, shipRotation.z, shipRotation.w);
    
    // Forward/backward thrust in ship's local forward direction
    let thrustZ = 0;
    if (shipMovement.forward) thrustZ -= 1;
    if (shipMovement.backward) thrustZ += 1;
    
    if (thrustZ !== 0) {
      const forwardVector = new THREE.Vector3(0, 0, thrustZ).applyQuaternion(shipQuat);
      const thrustImpulse = {
        x: forwardVector.x * thrustPower * deltaTime,
        y: forwardVector.y * thrustPower * deltaTime,
        z: forwardVector.z * thrustPower * deltaTime
      };
      shipBody.applyImpulse(thrustImpulse, true);
    }
    
    // Strong vertical lift (counter-gravity when up arrow pressed)
    if (shipMovement.up) {
      const strongLiftPower = 400; // Much stronger than gravity
      const liftImpulse = { x: 0, y: strongLiftPower * deltaTime, z: 0 };
      shipBody.applyImpulse(liftImpulse, true);
      console.log('Applying strong lift impulse:', liftImpulse);
    }
    
    // Yaw rotation (left/right arrows)
    let yawTorque = 0;
    if (shipMovement.left) yawTorque += 1;
    if (shipMovement.right) yawTorque -= 1;
    
    if (yawTorque !== 0) {
      const torqueImpulse = { x: 0, y: yawTorque * torquePower * deltaTime, z: 0 };
      shipBody.applyTorqueImpulse(torqueImpulse, true);
    }
    
    // Update ship mesh position and rotation to match physics
    const pos = shipBody.translation();
    const rot = shipBody.rotation();
    shipMesh.position.set(pos.x, pos.y, pos.z);
    shipMesh.quaternion.set(rot.x, rot.y, rot.z, rot.w);
    
    // Interior proxy always stays at origin (static)
    if (isInsideShip) {
      shipInteriorProxy.position.set(0, 0, 0);
      interiorCamera.lookAt(0, 1.5, 0);
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
      // Step exterior world
      exteriorWorld.timestep = deltaTime;
      exteriorWorld.step(exteriorEventQueue);
      
      // Process exterior collision events for ship entry detection
      exteriorEventQueue.drainCollisionEvents((handle1, handle2, started) => {
        if (started && !isInsideShip && !isTransitioning) {
          const collider1 = exteriorWorld.getCollider(handle1);
          const collider2 = exteriorWorld.getCollider(handle2);
          
          // Check if one of the colliders is the exterior player
          if ((collider1 === playerCollider || collider2 === playerCollider)) {
            const otherCollider = collider1 === playerCollider ? collider2 : collider1;
            
            // Check if colliding with door sensor
            if (otherCollider.isSensor()) {
              const playerPos = playerBody.translation();
              console.log('Exterior player detected at door sensor at position:', playerPos);
              console.log('Starting entry transition');
              startEnterTransition();
            }
          }
        }
      });
      
      // Step interior world
      interiorWorld.timestep = deltaTime;
      interiorWorld.step(interiorEventQueue);
      
      // Check for automatic exit based on momentum and clear path
      if (isInsideShip && !isTransitioning) {
        checkAutoExit();
      }
      
      // Process interior collision events for sensor detection
      interiorEventQueue.drainCollisionEvents((handle1, handle2, started) => {
        // Remove sensor-based exit detection - now using momentum + raycast system
      });
      
      // Update transition progress
      if (isTransitioning) {
        updateTransition(deltaTime);
      }
      
      updatePlayer(deltaTime);
      updateShip(deltaTime);
    }
    
    renderDualView();
  }
  
  function renderDualView() {
    const width = window.innerWidth;
    const height = window.innerHeight;
    
    // Clear the entire screen
    renderer.setScissorTest(false);
    renderer.clear();
    
    // Left viewport - Exterior world
    renderer.setViewport(0, 0, width / 2, height);
    renderer.setScissor(0, 0, width / 2, height);
    renderer.setScissorTest(true);
    
    // Show exterior ship and hide interior proxy
    shipMesh.visible = true;
    shipInteriorProxy.visible = false;
    // Always show player mesh in exterior world (positioned correctly whether inside or outside ship)
    playerMesh.visible = true;
    
    renderer.render(exteriorScene, exteriorCamera);
    
    // Right viewport - Interior world view
    renderer.setViewport(width / 2, 0, width / 2, height);
    renderer.setScissor(width / 2, 0, width / 2, height);
    
    // Show interior proxy and hide exterior ship
    shipMesh.visible = false;
    shipInteriorProxy.visible = true;
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
    
    // Reset viewport and scissor
    renderer.setViewport(0, 0, width, height);
    renderer.setScissorTest(false);
    
    // Restore player visibility for next frame
    playerMesh.visible = true;
  }
  
  function onWindowResize() {
    // Update aspect ratio for split screen (half width)
    exteriorCamera.aspect = (window.innerWidth / 2) / window.innerHeight;
    exteriorCamera.updateProjectionMatrix();
    interiorCamera.aspect = (window.innerWidth / 2) / window.innerHeight;
    interiorCamera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
  }
</script>

<canvas bind:this={canvas}></canvas>

<div class="ui">
  <p>WASD: Move | Space: Jump | Arrow Keys: Fly Ship | Q/E: Up/Down</p>
  <p>Status: {isInsideShip ? 'Inside Ship' : 'Outside Ship'}</p>
  <p>Movement: {movementStatus}</p>
</div>

<div class="camera-labels">
  <div class="camera-label left">Exterior World</div>
  <div class="camera-label right">Interior Proxy</div>
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
  
  .camera-label.right {
    right: 20px;
    color: #00aaff;
  }
</style>