<template>
  <v-app>
    <v-main>
      <div class="game-container">
        <!-- Log des morts -->
        <div class="death-logs" v-if="deathLogs">
          <div class="death-log">{{ deathLogs }}</div>
        </div>

        <div id="world"></div>

        <!-- Overlay avatar quand un joueur touche le sol -->
        <div v-if="featuredAvatar" class="featured-avatar" @click="featuredAvatar = null">
          <div class="avatar-wrapper">
            <img :src="featuredAvatar.image" :alt="featuredAvatar.username" />
          </div>
          <div class="featured-username">@{{ featuredAvatar.username }}</div>
        </div>
        <button @click="dropAvatars()" class="drop-button">Lancer les avatars</button>
        <!-- Ajout du rendu du monde -->
      </div>
    </v-main>
  </v-app>
</template>

<script>
import Matter from "matter-js";

// Génère un username aléatoire lisible
function generateUsername() {
  const adjectives = [
    "Swift",
    "Brave",
    "Lucky",
    "Mighty",
    "Silent",
    "Fuzzy",
    "Icy",
    "Cosmic",
    "Crimson",
    "Neon",
    "Turbo",
    "Shadow",
    "Golden",
    "Pixel",
    "Magic",
    "Witty",
    "Funky",
    "Rapid",
    "Noble",
    "Wild",
  ];
  const nouns = [
    "Falcon",
    "Panda",
    "Wizard",
    "Ninja",
    "Comet",
    "Tiger",
    "Dragon",
    "Otter",
    "Phoenix",
    "Wolf",
    "Raccoon",
    "Cobra",
    "Golem",
    "Unicorn",
    "Pirate",
    "Samurai",
    "Knight",
    "Panther",
    "Meteor",
    "Goblin",
  ];
  const adj = adjectives[Math.floor(Math.random() * adjectives.length)];
  const noun = nouns[Math.floor(Math.random() * nouns.length)];
  const number = Math.floor(Math.random() * 999)
    .toString()
    .padStart(2, "0");
  return `${adj}${noun}${number}`;
}

export default {
  name: "App",
  data() {
    return {
      spriteSize: 12, // Nouvelle variable pour régler la taille des sprites
      pegSize: 5, // Nouvelle variable pour régler la taille des pegs
      pegCount: 132, // Nouvelle variable pour régler le nombre de pegs
      avatars: Array.from({ length: 10 }, (_, i) => ({
        id: i + 1,
        image: `https://i.pravatar.cc/150?img=${Math.floor(Math.random() * 70) + 1}`,
        health: 4,
        username: generateUsername(),
      })),
      pegs: Array.from({ length: 20 }, (_, i) => ({
        id: i + 1,
        style: {
          left: `${Math.random() * 90 + 5}%`,
          top: `${Math.random() * 70 + 10}%`,
        },
      })),
      poucentageBomb: 20, // Nouvelle variable pour régler le pourcentage de bombes
      buckets: [
        { id: 1, width: 20, pos: 20, win: false },
        { id: 2, width: 20, pos: 50, win: true }, // Ajout bac au milieu
        { id: 3, width: 20, pos: 80, win: false }, // Ajout bac en bas à droite
      ],
      engine: null,
      render: null,
      world: null,
      worldWidth: 450, // Updated width of the world
      worldHeight: 800, // Updated height of the world
      ground: null,
      featuredAvatar: null, // Avatar mis en avant quand il touche le sol
      colors: ["#F44336", "#2196F3", "#4CAF50", "#FFC107", "#673AB7", "#FF9800"], // Tableau de couleurs
      deathLogs: "", // Nouveau tableau pour stocker les logs des morts
    };
  },
  methods: {
    initializeObstacles() {
      // Génère des obstacles statiques (pegs) dans le monde
      const pegs = [];
      const rows = Math.ceil(this.pegCount / 8); // Calcul du nombre de rangées en fonction du nombre de pegs
      const cols = 11; // Nombre de colonnes d'obstacles
      const spacingX = this.worldWidth / cols - 1; // Espacement horizontal entre les obstacles
      const spacingY = 50; // Espacement vertical entre les obstacles
      const offsetX = 15; // Décalage horizontal initial
      const offsetY = 100; // Décalage vertical initial
      const colors = this.colors; // Récupère le tableau de couleurs

      for (let row = 0; row < rows; row++) {
        for (let col = 0; col < cols; col++) {
          if (pegs.length >= this.pegCount) break; // Arrête si le nombre de pegs est atteint

          const x = offsetX + col * spacingX + (row % 2 === 0 ? spacingX / 2 : 0); // Décalage pour les rangées impaires
          const y = offsetY + row * spacingY;
          const isTrapped = Math.random() < this.poucentageBomb / 100; // 20% de chance que le peg soit piégé
          const randomColor = colors[Math.floor(Math.random() * colors.length)]; // Couleur aléatoire
          const peg = Matter.Bodies.circle(x, y, this.pegSize, {
            isStatic: true,
            render: isTrapped
              ? {
                  sprite: {
                    texture: "src/assets/bomb.png", // Chemin local vers l'icône
                    xScale: this.pegSize / 50, // Ajustez l'échelle selon vos besoins
                    yScale: this.pegSize / 50,
                  },
                }
              : {
                  fillStyle: randomColor, // Applique une couleur aléatoire
                  opacity: 0.8,
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
              this.deathLogs = `@${avatar.username} est éliminé.`;
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

    getLeftPos(bucket) {
      //get all widths of bucket befores
      let indexBucket = this.buckets.findIndex((b) => b.id === bucket.id);
      let totalWidth = 0;
      for (let i = 0; i < indexBucket; i++) {
        totalWidth += this.worldWidth * (this.buckets[i].width / 100);
      }
      return totalWidth;
    },

    initializePhysics() {
      this.engine = Matter.Engine.create();
      this.world = this.engine.world;

      // Réduction de la gravité pour ralentir la chute
      this.engine.world.gravity.y = 0.07; // Valeur par défaut est 1, ici on la réduit à 0.5

      // Ajoute les murs du plateau
      const ground = Matter.Bodies.rectangle(this.worldWidth / 2, this.worldHeight, this.worldWidth, 50, {
        isStatic: true,
        label: "Ground",
        render: {
          fillStyle: "", // Couleur du sol
        },
      }); // Updated ground dimensions
      const leftWall = Matter.Bodies.rectangle(0, this.worldHeight / 2, 1, this.worldHeight, { isStatic: true }); // Updated left wall dimensions
      const rightWall = Matter.Bodies.rectangle(this.worldWidth, this.worldHeight / 2, 1, this.worldHeight, {
        isStatic: true,
      }); // Updated right wall dimensions
      Matter.World.add(this.world, [ground, leftWall, rightWall]);
      this.ground = ground;

      // Ajoute les obstacles
      this.initializeObstacles();

      // Ajoute les buckets au monde Matter.js
      this.buckets.forEach((bucket, i) => {
        const bucketBody = Matter.Bodies.rectangle(
          this.worldWidth * (bucket.pos / 100),
          this.worldHeight - 50, // Position verticale en bas
          this.worldWidth * (bucket.width / 100),
          20, // Hauteur du bucket
          {
            isStatic: true,
            label: `Bucket-${bucket.id}`,
            render: {
              sprite: {
                texture: bucket.win ? "src/assets/win.png" : "src/assets/dead.png", // Chemin local vers l'icône
                xScale: this.pegSize / 50, // Ajustez l'échelle selon vos besoins
                yScale: this.pegSize / 50,
              },
            },
          }
        );
        Matter.World.add(this.world, bucketBody);
        bucket.body = bucketBody; // Associe le corps physique au bucket
      });

      // Prépare les images (cache) et ajoute les avatars comme cercles physiques
      this.avatars.forEach((avatar) => {
        // Création du corps physique circulaire (sans sprite direct)
        const body = Matter.Bodies.circle(Math.random() * (this.worldWidth - 50) + 25, 50, this.spriteSize / 2, {
          restitution: 0.8,
          render: {
            fillStyle: "rgba(0,0,0,0)", // Rendre le cercle invisible (on dessine nous‑mêmes)
            lineWidth: 0,
          },
          label: "Avatar",
        });
        avatar.body = body;
        // Charge l'image et la met en cache
        const img = new Image();
        // crossOrigin retiré pour éviter des blocages éventuels CORS avec pravatar
        img.src = avatar.image;
        img.onload = () => {
          avatar._ready = true; /* console.log('Image ok', avatar.id);*/
        };
        img.onerror = () => {
          avatar._error = true;
        };
        avatar._cachedImage = img;
        Matter.World.add(this.world, body);

        // Crée un élément DOM pour afficher les cœurs
        const heartElement = document.createElement("div");
        heartElement.className = "hearts";
        heartElement.id = `hearts-${avatar.id}`;
        heartElement.style.position = "absolute";
        heartElement.style.transform = `translate(-50%, -100%)`;
        heartElement.style.fontSize = "3px";
        heartElement.innerHTML = "❤️".repeat(avatar.health);
        document.getElementById("world").appendChild(heartElement);
      });

      // Ajoute le rendu Matter.js au conteneur #world
      this.render = Matter.Render.create({
        element: document.getElementById("world"),
        engine: this.engine,
        options: {
          width: this.worldWidth, // Largeur dynamique
          height: this.worldHeight, // Hauteur dynamique
          wireframes: false,
        },
      });
      Matter.Render.run(this.render);

      // Dessin personnalisé pour rendre les avatars ronds (masque circulaire)
      Matter.Events.on(this.render, "afterRender", () => {
        const ctx = this.render.context;
        this.avatars.forEach((avatar) => {
          if (!avatar.body) return;
          const radius = this.spriteSize / 2;
          const { x, y } = avatar.body.position;
          const img = avatar._cachedImage;

          ctx.save();
          ctx.beginPath();
          ctx.arc(x, y, radius, 0, Math.PI * 2);
          ctx.closePath();
          ctx.clip();
          ctx.drawImage(img, x - radius, y - radius, radius * 2, radius * 2);
          ctx.restore();
        });
      });

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

      // Positionne les avatars aléatoirement dans une zone de 50px de hauteur
      this.avatars.forEach((avatar) => {
        const randomY = Math.random() * 50; // Génère une position Y aléatoire dans une plage de 50px
        Matter.Body.setPosition(avatar.body, {
          x: Math.random() * (this.worldWidth - 50) + 25, // Position X aléatoire
          y: randomY - 50, // Position Y aléatoire
        });
      });
    },
    monitorCollisions() {
      Matter.Events.on(this.engine, "collisionStart", (event) => {
        const pairs = event.pairs;

        pairs.forEach((pair) => {
          const bodyA = pair.bodyA;
          const bodyB = pair.bodyB;

          const avatarA = this.avatars.find((avatar) => avatar.body === bodyA);
          const avatarB = this.avatars.find((avatar) => avatar.body === bodyB);

          const bucketA = this.buckets.find((bucket) => bucket.body === bodyA);
          const bucketB = this.buckets.find((bucket) => bucket.body === bodyB);

          if (avatarA && bucketB && bucketB.win && !avatarA._touchedWin) {
            avatarA._touchedWin = true;
            if (!this.featuredAvatar) this.featuredAvatar = avatarA; // Set the winner
          } else if (avatarB && bucketA && bucketA.win && !avatarB._touchedWin) {
            avatarB._touchedWin = true;
            if (!this.featuredAvatar) this.featuredAvatar = avatarB; // Set the winner
          }

          if (avatarA && bucketB && !bucketB.win) {
            const logMessage = `${avatarA.username} est mort.`;
            console.log(logMessage);
            Matter.World.remove(this.world, avatarA.body);
            this.avatars = this.avatars.filter((a) => a !== avatarA);
            const heartElement = document.getElementById(`hearts-${avatarA.id}`);
            if (heartElement) {
              heartElement.remove();
            }
          } else if (avatarB && bucketA && !bucketA.win) {
            const logMessage = `${avatarB.username} est mort.`;
            console.log(logMessage);
            Matter.World.remove(this.world, avatarB.body);
            this.avatars = this.avatars.filter((a) => a !== avatarB);
            const heartElement = document.getElementById(`hearts-${avatarB.id}`);
            if (heartElement) {
              heartElement.remove();
            }
          }

          // Détection avatar + sol
          //   const groundTouchedByA = avatarA && (bodyB === this.ground || bodyB.label === "Ground");
          //   const groundTouchedByB = avatarB && (bodyA === this.ground || bodyA.label === "Ground");
          //   if (groundTouchedByA && !avatarA._touchedGround) {
          //     avatarA._touchedGround = true;
          //     if (!this.featuredAvatar) this.featuredAvatar = avatarA; // ne remplace pas si déjà défini
          //   } else if (groundTouchedByB && !avatarB._touchedGround) {
          //     avatarB._touchedGround = true;
          //     if (!this.featuredAvatar) this.featuredAvatar = avatarB; // ne remplace pas si déjà défini
          //   }
        });
      });
    },
  },
};
</script>

<style lang="scss">
.game-container {
  text-align: center;
  position: relative;
}

.death-logs {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  font-size: 14px;
  padding: 10px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.2);
  z-index: 10;
}

.death-log {
  margin-bottom: 5px;
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
  height: 800px;
  background-color: #f0f0f0;
  //   border: 2px solid #ccc;
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
  background-color: red; /* Fond rouge pour les buckets */
  //   border: 1px solid #aaa;
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
  width: 450px; /* Largeur adaptée pour un écran de téléphone */
  height: 800px; /* Hauteur adaptée pour un écran de téléphone */
  background-color: #e0e0e0; /* Couleur de fond pour visualiser le conteneur */
  //   border: 1px solid #ccc; /* Bordure pour mieux le distinguer */
  position: relative;
}

.hearts {
  position: absolute;
  pointer-events: none; /* Ignore les événements de souris */
  font-size: 5px; /* Taille de police pour les cœurs */
}

.featured-avatar {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(0, 0, 0, 0.6);
  padding: 20px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 20;
  cursor: pointer;
  animation: popin 0.35s ease;
  flex-direction: column;
}

.featured-avatar img {
  width: 220px;
  height: 220px;
  border-radius: 50%;
  object-fit: cover;
  border: 6px solid #fff;
  box-shadow: 0 4px 18px rgba(0, 0, 0, 0.4);
}

.featured-username {
  margin-top: 14px;
  font-size: 22px;
  font-weight: 600;
  font-family: system-ui, "Segoe UI", Roboto, Arial, sans-serif;
  color: #fff;
  text-shadow: 0 2px 4px rgba(0, 0, 0, 0.5);
  letter-spacing: 0.5px;
}

@keyframes popin {
  0% {
    transform: translate(-50%, -50%) scale(0.6);
    opacity: 0;
  }
  100% {
    transform: translate(-50%, -50%) scale(1);
    opacity: 1;
  }
}
</style>
