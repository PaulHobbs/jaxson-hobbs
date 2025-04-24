<script lang="ts">
  import { onMount, onDestroy } from 'svelte';

  let game: Phaser.Game;


  onMount(async () => {
    const Phaser = await import('phaser');

    class FastRunnerScene extends Phaser.Scene {
      player!: Phaser.Physics.Arcade.Sprite;
      cursors!: Phaser.Types.Input.Keyboard.CursorKeys;
      platforms!: Phaser.Physics.Arcade.StaticGroup;
      rings!: Phaser.Physics.Arcade.Group; // Declare rings group
      hazards!: Phaser.Physics.Arcade.StaticGroup; // Declare hazards group
      score!: number; // Declare score property
      scoreText!: Phaser.GameObjects.Text; // Declare scoreText property
      backgroundFar!: Phaser.GameObjects.TileSprite; // Declare far background property
      backgroundMid!: Phaser.GameObjects.TileSprite; // Declare mid background property
      music!: Phaser.Sound.BaseSound; // Declare music property

      preload() {
        this.textures.generate('player', { data: ['1'], pixelWidth: 32 });
        this.textures.generate('ground', { data: ['#'.repeat(64)], pixelWidth: 64 });
        // Generate a simple yellow circle texture for the ring
        this.textures.generate('ring', { data: ['E'], pixelWidth: 16, palette: { E: '#ffff00' } as any });
        // Generate a simple red square texture for the hazard
        this.textures.generate('hazard', { data: ['2'], pixelWidth: 32, palette: { '2': '#ff0000' } as any });

        // Generate placeholder textures for parallax backgrounds
        this.textures.generate('background-far', { data: ['+'], pixelWidth: 1, palette: { '+': '#334455' } as any });
        this.textures.generate('background-mid', { data: ['.'], pixelWidth: 1, palette: { '.': '#556677' } as any });

        // Load placeholder audio files
        this.load.audio('bgm', 'assets/audio/music.ogg');
        this.load.audio('sfx_jump', 'assets/audio/jump.ogg');
        this.load.audio('sfx_ring', 'assets/audio/ring.ogg');
        this.load.audio('sfx_hit', 'assets/audio/hit.ogg');
      }

      create() {
        const gameWidth = this.sys.game.config.width as number;
        const gameHeight = this.sys.game.config.height as number;

        // Add parallax background layers
        this.backgroundFar = this.add.tileSprite(0, 0, gameWidth, gameHeight, 'background-far').setOrigin(0, 0).setScrollFactor(0.25);
        this.backgroundMid = this.add.tileSprite(0, 0, gameWidth, gameHeight, 'background-mid').setOrigin(0, 0).setScrollFactor(0.5);

        const centerX = gameWidth / 2;
        const centerY = gameHeight / 2;
        this.player = this.physics.add.sprite(centerX, centerY, 'player');
        this.player.setCollideWorldBounds(true);

        const startX = centerX; // Define startX
        const startY = centerY; // Define startY

        this.platforms = this.physics.add.staticGroup();

        // Add platforms
        this.platforms.create(400, 568, 'ground').setScale(2).refreshBody(); // Ground platform
        this.platforms.create(600, 400, 'ground'); // Elevated platform 1
        this.platforms.create(50, 250, 'ground'); // Elevated platform 2
        this.platforms.create(750, 220, 'ground'); // Elevated platform 3

        // Add collider between player and platforms
        this.physics.add.collider(this.player, this.platforms);

        // Create hazards group
        this.hazards = this.physics.add.staticGroup();

        // Add hazards
        this.hazards.create(300, 536, 'hazard'); // Hazard on ground platform
        this.hazards.create(700, 368, 'hazard'); // Hazard on elevated platform

        // Add collider between player and hazards
        this.physics.add.collider(this.player, this.hazards, this.hitHazard, undefined, this);

        // Create rings group
        this.rings = this.physics.add.group({ allowGravity: false });

        // Add rings
        this.rings.create(100, 500, 'ring');
        this.rings.create(150, 500, 'ring');
        this.rings.create(600, 350, 'ring');

        // Set world and camera bounds (optional but recommended)
        this.physics.world.setBounds(0, 0, 1600, 600);
        this.cameras.main.setBounds(0, 0, 1600, 600);

        // Make the camera follow the player
        this.cameras.main.startFollow(this.player);

        // Initialize score
        this.score = 0;

        // Create score text
        this.scoreText = this.add.text(16, 16, 'Score: 0', { fontSize: '32px', color: '#FFF' });
        this.scoreText.setScrollFactor(0); // Make score text fixed to camera

        // Add overlap detector between player and rings
        this.physics.add.overlap(this.player, this.rings, this.collectRing, undefined, this);

        this.cursors = this.input.keyboard!.createCursorKeys();

        // Add and play background music if loaded
        if (this.sound.get('bgm')) {
          this.music = this.sound.add('bgm', { loop: true, volume: 0.5 });
          this.music.play();
        }
      }

      // Callback function to collect rings
      collectRing(player: Phaser.Types.Physics.Arcade.GameObjectWithBody | Phaser.Tilemaps.Tile | Phaser.Physics.Arcade.Body | Phaser.Physics.Arcade.StaticBody, ring: Phaser.Types.Physics.Arcade.GameObjectWithBody | Phaser.Tilemaps.Tile | Phaser.Physics.Arcade.Body | Phaser.Physics.Arcade.StaticBody) {
        // Ensure both objects are sprites before proceeding
        if (player instanceof Phaser.Physics.Arcade.Sprite && ring instanceof Phaser.Physics.Arcade.Sprite) {
          ring.disableBody(true, true);
          // Increment score and update text
          this.score += 10;
          this.scoreText.setText('Score: ' + this.score);
          // Play ring collection sound effect if loaded
          if (this.sound.get('sfx_ring')) {
            this.sound.play('sfx_ring');
          }
        }
      }

      // Callback function when player hits a hazard
      hitHazard(player: Phaser.Types.Physics.Arcade.GameObjectWithBody | Phaser.Tilemaps.Tile | Phaser.Physics.Arcade.Body | Phaser.Physics.Arcade.StaticBody, hazard: Phaser.Types.Physics.Arcade.GameObjectWithBody | Phaser.Tilemaps.Tile | Phaser.Physics.Arcade.Body | Phaser.Physics.Arcade.StaticBody) {
        // Ensure both objects are sprites before proceeding
        if (player instanceof Phaser.Physics.Arcade.Sprite && hazard instanceof Phaser.Physics.Arcade.Sprite) {
          player.setTint(0xff0000);
          this.time.delayedCall(100, () => { player.clearTint(); });
          // Play hit hazard sound effect if loaded
          if (this.sound.get('sfx_hit')) {
            this.sound.play('sfx_hit');
          }
        }
      }

      update() {
        if (this.cursors.left.isDown) {
          this.player.setVelocityX(-160);
        } else if (this.cursors.right.isDown) {
          this.player.setVelocityX(160);
        } else {
          this.player.setVelocityX(0);
        }

        if (this.player.body && (this.cursors.space.isDown || this.cursors.up.isDown) && this.player.body.touching.down) {
          this.player.setVelocityY(-330);
          // Play jump sound effect if loaded
          if (this.sound.get('sfx_jump')) {
            this.sound.play('sfx_jump');
          }
        }

        // Update tile positions for parallax effect
        this.backgroundFar.tilePositionX = this.cameras.main.scrollX * 0.25;
        this.backgroundMid.tilePositionX = this.cameras.main.scrollX * 0.5;
      }
    }

    const config: Phaser.Types.Core.GameConfig = {
      type: Phaser.AUTO,
      width: 800,
      height: 600,
      parent: 'phaser-game-container',
      backgroundColor: '#2d2d2d',
      scene: FastRunnerScene,
      physics: {
        default: 'arcade',
        arcade: {
          gravity: { x: 0, y: 300 },
          debug: false
        }
      }
    };

    game = new Phaser.Game(config);
  });

  onDestroy(() => {
    if (game) {
      game.destroy(true);
    }
  });
</script>

<div id="phaser-game-container">
  <!-- Phaser game canvas will be injected here -->
</div>

<style>
  /* Styles go here */
</style>