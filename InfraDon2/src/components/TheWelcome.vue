<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';

// Interface pour un jeu
interface Game {
  _id: string;
  _rev?: string;
  biblio: {
    games: Array<{
      title: string;
      editor: string;
      country?: string;
      release: number;
    }>;
  };
}

// Référence à la base de données
const storage = ref<PouchDB.Database>();
// Données stockées (liste de jeux)
const gamesData = ref<Game[]>([]);
// Données du formulaire
const newGame = ref({
  title: '',
  editor: '',
  country: '',
  release: new Date().getFullYear(),
});
// ID du jeu en cours d'édition
const editingId = ref<string | null>(null);

// Initialisation de la base de données
const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:admin@localhost:5984/infradon2');
  if (db) {
    console.log("Connecté à la collection : " + db.name);
    storage.value = db;
  } else {
    console.warn('Échec lors de la connexion à la base de données');
  }
};

// Récupération des données
const fetchData = async () => {
  if (!storage.value) return;
  try {
    const result = await storage.value.allDocs({ include_docs: true });
    gamesData.value = result.rows
      .map(row => row.doc as Game)
      .filter(doc => doc.biblio && doc.biblio.games);
    console.log('Données récupérées :', gamesData.value);
  } catch (error) {
    console.error('Erreur lors de la récupération des données :', error);
  }
};

// Fonction pour ajouter un jeu
const addGame = async (title: string, editor: string, country?: string, release?: number) => {
  if (!storage.value) {
    console.warn('Base de données non initialisée');
    return;
  }

  const newDocId = `game_${Date.now()}`;
  const newGameDoc: Game = {
    _id: newDocId,
    biblio: {
      games: [
        {
          title,
          editor,
          country,
          release: release || new Date().getFullYear(),
        },
      ],
    },
  };

  try {
    const response = await storage.value.put(newGameDoc);
    console.log('Document ajouté avec succès :', response);
    await fetchData();
  } catch (error) {
    console.error('Erreur lors de l\'ajout du jeu :', error);
  }
};

// Fonction pour mettre à jour un jeu
const updateGame = async (id: string, title: string, editor: string, country?: string, release?: number) => {
  if (!storage.value) {
    console.warn('Base de données non initialisée');
    return;
  }

  try {
    // Récupère le document actuel pour obtenir le _rev
    const doc = await storage.value.get(id);
    // Met à jour les données du jeu
    doc.biblio.games[0] = {
      title,
      editor,
      country,
      release: release || new Date().getFullYear(),
    };

    // Sauvegarde les modifications
    const response = await storage.value.put(doc);
    console.log('Document mis à jour avec succès :', response);

    // Réinitialise l'état d'édition
    editingId.value = null;
    newGame.value = {
      title: '',
      editor: '',
      country: '',
      release: new Date().getFullYear(),
    };

    // Rafraîchit la liste des jeux
    await fetchData();
  } catch (error) {
    console.error('Erreur lors de la mise à jour du jeu :', error);
  }
};

// Soumettre le formulaire (ajout ou mise à jour)
const submitNewGame = () => {
  if (editingId.value) {
    // Si on est en mode édition, on met à jour le jeu
    updateGame(
      editingId.value,
      newGame.value.title,
      newGame.value.editor,
      newGame.value.country,
      newGame.value.release
    );
  } else {
    // Sinon, on ajoute un nouveau jeu
    addGame(
      newGame.value.title,
      newGame.value.editor,
      newGame.value.country,
      newGame.value.release
    );
  }
};

// Remplir le formulaire avec les données du jeu à modifier
const editGame = (game: Game) => {
  editingId.value = game._id;
  newGame.value = {
    title: game.biblio.games[0].title,
    editor: game.biblio.games[0].editor,
    country: game.biblio.games[0].country || '',
    release: game.biblio.games[0].release,
  };
};

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase();
  fetchData();
});
</script>

<template>
  <h1>Games List</h1>

  <!-- Formulaire pour ajouter/mettre à jour un jeu -->
  <form @submit.prevent="submitNewGame" class="game-form">
    <h2>{{ editingId ? 'Modifier un jeu' : 'Ajouter un jeu' }}</h2>
    <div>
      <label for="title">Titre :</label>
      <input id="title" v-model="newGame.title" placeholder="Titre du jeu" required />
    </div>
    <div>
      <label for="editor">Éditeur :</label>
      <input id="editor" v-model="newGame.editor" placeholder="Éditeur" required />
    </div>
    <div>
      <label for="country">Pays (optionnel) :</label>
      <input id="country" v-model="newGame.country" placeholder="Pays" />
    </div>
    <div>
      <label for="release">Année de sortie :</label>
      <input id="release" v-model.number="newGame.release" type="number" placeholder="Année" required />
    </div>
    <button type="submit">{{ editingId ? 'Mettre à jour' : 'Ajouter le jeu' }}</button>
    <button v-if="editingId" type="button" @click="editingId = null; newGame = { title: '', editor: '', country: '', release: new Date().getFullYear() }">
      Annuler
    </button>
  </form>

  <!-- Liste des jeux -->
  <div v-if="gamesData.length > 0">
    <h2>Liste des jeux</h2>
    <div v-for="game in gamesData" :key="game._id" class="game-card">
      <div v-for="(g, index) in game.biblio.games" :key="index">
        <h3>{{ g.title }}</h3>
        <p><strong>Éditeur :</strong> {{ g.editor }}</p>
        <p v-if="g.country"><strong>Pays :</strong> {{ g.country }}</p>
        <p><strong>Année de sortie :</strong> {{ g.release }}</p>
        <button @click="editGame(game)">Modifier</button>
      </div>
    </div>
  </div>
  <p v-else>Aucun jeu trouvé.</p>
</template>