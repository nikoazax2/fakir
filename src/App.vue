<template>
  <v-app>
    <v-main>
      <div class="game-container">
        <!-- <h1>Fakir des Profils Concept</h1> -->

        <div id="world"></div>
        <button @click="dropAvatars()" class="drop-button">Lancer les avatars</button>
        <!-- Ajout du rendu du monde -->
      </div>
    </v-main>
  </v-app>
</template>

<script>
import Matter from "matter-js";

export default {
  name: "App",
  data() {
    return {
      spriteSize: 10, // Nouvelle variable pour régler la taille des sprites
      avatars: Array.from({ length: 50 }, (_, i) => ({
        id: i + 1,
        image: `https://i.pravatar.cc/150?img=${i + 1}`,
        health: 4,
      })),
      pegs: Array.from({ length: 20 }, (_, i) => ({
        id: i + 1,
        style: {
          left: `${Math.random() * 90 + 5}%`,
          top: `${Math.random() * 70 + 10}%`,
        },
      })),
      buckets: [
        { id: 1, effect: "❤️", style: { left: "10%" } },
        { id: 2, effect: "⚡", style: { left: "30%" } },
        { id: 3, effect: "💀", style: { left: "50%" } },
        { id: 4, effect: "🎁", style: { left: "70%" } },
        { id: 5, effect: "👑", style: { left: "90%" } },
      ],
      engine: null,
      render: null,
      world: null,
    };
  },
  methods: {
    initializeObstacles() {
      // Génère des obstacles statiques (pegs) dans le monde
      const pegs = [];
      const rows = 10; // Nombre de rangées d'obstacles
      const cols = 8; // Nombre de colonnes d'obstacles
      const spacingX = 40; // Espacement horizontal entre les obstacles
      const spacingY = 50; // Espacement vertical entre les obstacles
      const offsetX = 35; // Décalage horizontal initial
      const offsetY = 100; // Décalage vertical initial

      for (let row = 0; row < rows; row++) {
        for (let col = 0; col < cols; col++) {
          const x = offsetX + col * spacingX + (row % 2 === 0 ? spacingX / 2 : 0); // Décalage pour les rangées impaires
          const y = offsetY + row * spacingY;
          const isTrapped = Math.random() < 0.5; // 20% de chance que le peg soit piégé
          const peg = Matter.Bodies.circle(x, y, 5, {
            isStatic: true,
            render: {
              fillStyle: isTrapped ? "red" : "black", // Couleur rouge pour les pegs piégés
            },
            label: "Peg", // Ajout d'un label pour identifier les pegs
          });
          peg.isTrapped = isTrapped; // Marqueur pour identifier les pegs piégés
          pegs.push(peg);
        }
      }

      Matter.World.add(this.world, pegs);
      this.pegs = pegs; // Mise à jour de this.pegs avec tous les pegs générés

      // Ajoute un écouteur pour détecter les collisions avec les pegs piégés
      Matter.Events.on(this.engine, "collisionStart", (event) => {
        const pairs = event.pairs;

        pairs.forEach((pair) => {

          const bodyA = pair.bodyA;
          const bodyB = pair.bodyB;

          const trappedPeg = this.pegs.find((peg) => peg === bodyA || peg === bodyB);
          const avatar = this.avatars.find((avatar) => avatar.body === bodyA || avatar.body === bodyB);

          if (trappedPeg && trappedPeg.isTrapped && avatar) { 
            avatar.health -= 1; // Réduit la vie de l'avatar

            // Supprime l'avatar si sa vie atteint 0
            if (avatar.health <= 0) {
              console.log(`Avatar ${avatar.id} est éliminé.`);
              Matter.World.remove(this.world, avatar.body);
              this.avatars = this.avatars.filter((a) => a !== avatar);
              const heartElement = document.getElementById(`hearts-${avatar.id}`);
              if (heartElement) {
                heartElement.remove();
              }
            }
          }
        });
      });
    },

    initializePhysics() {
      this.engine = Matter.Engine.create();
      this.world = this.engine.world;

      // Réduction de la gravité pour ralentir la chute
      this.engine.world.gravity.y = 0.1; // Valeur par défaut est 1, ici on la réduit à 0.5

      // Ajoute les murs du plateau
      const ground = Matter.Bodies.rectangle(187.5, 667, 375, 50, { isStatic: true }); // Mur du bas
      const leftWall = Matter.Bodies.rectangle(0, 333.5, 50, 667, { isStatic: true }); // Mur de gauche
      const rightWall = Matter.Bodies.rectangle(375, 333.5, 50, 667, { isStatic: true }); // Mur de droite
      Matter.World.add(this.world, [ground, leftWall, rightWall]);

      // Ajoute les obstacles
      this.initializeObstacles();

      // Ajoute les avatars comme des cercles physiques avec des images
      this.avatars.forEach((avatar, index) => {
        const body = Matter.Bodies.circle(Math.random() * 325 + 25, 50, this.spriteSize / 2, {
          restitution: 0.8,
          render: {
            sprite: {
              texture: avatar.image,
              xScale: this.spriteSize / 100, // Ajuste l'échelle en fonction de la taille
              yScale: this.spriteSize / 100, // Ajuste l'échelle en fonction de la taille
            },
          },
          label: "Avatar", // Ajout d'un label pour identifier les avatars
        });
        avatar.body = body;
        Matter.World.add(this.world, body);

        // Crée un élément DOM pour afficher les cœurs
        const heartElement = document.createElement("div");
        heartElement.className = "hearts";
        heartElement.id = `hearts-${avatar.id}`;
        heartElement.style.position = "absolute";
        heartElement.style.transform = `translate(-1%, -1%)`;
        heartElement.innerHTML = "❤️".repeat(avatar.health);
        document.getElementById("world").appendChild(heartElement);
      });

      // Ajoute le rendu Matter.js au conteneur #world
      this.render = Matter.Render.create({
        element: document.getElementById("world"),
        engine: this.engine,
        options: {
          width: 375, // Largeur adaptée pour un écran de téléphone
          height: 667, // Hauteur adaptée pour un écran de téléphone
          wireframes: false,
        },
      });
      Matter.Render.run(this.render);

      // Démarre le moteur physique avec le Runner
      const runner = Matter.Runner.create();
      Matter.Runner.run(runner, this.engine);

      // Met à jour les positions des cœurs
      Matter.Events.on(this.engine, "afterUpdate", () => {
        this.avatars.forEach((avatar) => {
          const heartElement = document.getElementById(`hearts-${avatar.id}`);
          if (heartElement) {
            heartElement.style.left = `${avatar.body.position.x}px`;
            heartElement.style.top = `${avatar.body.position.y - this.spriteSize}px`;
            heartElement.innerHTML = "❤️".repeat(avatar.health);
          }
        });
      });
    },
    dropAvatars() {
      this.initializePhysics();
      this.monitorCollisions(); // Ajout de la surveillance des collisions
    },
    monitorCollisions() {
      Matter.Events.on(this.engine, "collisionStart", (event) => {
        const pairs = event.pairs;

        pairs.forEach((pair) => {
          const bodyA = pair.bodyA;
          const bodyB = pair.bodyB;

          const avatarA = this.avatars.find((avatar) => avatar.body === bodyA);
          const avatarB = this.avatars.find((avatar) => avatar.body === bodyB);

          if (avatarA && avatarB) {
            // Suppression de la logique de perte de vie entre avatars
          }
        });
      });
    },
  },
};
</script>

<style lang="scss">
.game-container {
  text-align: center;
}

.avatars {
  display: flex;
  justify-content: center;
  flex-wrap: wrap;
  margin-bottom: 20px;
}

.avatar {
  width: 50px;
  height: 50px;
  border-radius: 50%;
  background-size: cover;
  background-position: center;
  margin: 5px;
}

.game-board {
  position: relative;
  width: 100%;
  height: 500px;
  background-color: #f0f0f0;
  border: 2px solid #ccc;
  border-radius: 10px;
  overflow: hidden;
}

.peg {
  position: absolute;
  width: 10px;
  height: 10px;
  background-color: #000;
  border-radius: 50%;
}

.bucket {
  position: absolute;
  bottom: 0;
  width: 50px;
  height: 50px;
  background-color: #ddd;
  border: 1px solid #aaa;
  text-align: center;
  line-height: 50px;
  border-radius: 5px;
}

.drop-button {
  margin: 20px auto;
  padding: 10px 20px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-size: 16px;
}

.drop-button:hover {
  background-color: #0056b3;
}

#world {
  width: 375px; /* Largeur adaptée pour un écran de téléphone */
  height: 667px; /* Hauteur adaptée pour un écran de téléphone */
  background-color: #e0e0e0; /* Couleur de fond pour visualiser le conteneur */
  border: 1px solid #ccc; /* Bordure pour mieux le distinguer */
}

.hearts {
  position: absolute;
  pointer-events: none; /* Ignore les événements de souris */
  font-size: 5px; /* Taille de police pour les cœurs */
}
</style>
