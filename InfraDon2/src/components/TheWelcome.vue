<script setup lang="ts">
import { onMounted, ref } from 'vue';
import PouchDB from 'pouchdb';
import PouchFind from 'pouchdb-find';

PouchDB.plugin(PouchFind);

// Interface
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

// Références
const storage = ref<PouchDB.Database>();
const gamesData = ref<Game[]>([]);

// Mode offline
const isOffline = ref(false);

// Formulaire
const newGame = ref({
  title: '',
  editor: '',
  country: '',
  release: new Date().getFullYear(),
});

const editingId = ref<string | null>(null);

// Recherche
const searchTitle = ref('');

const searchGames = async () => {
  if (!storage.value) return;

  if (!searchTitle.value.trim()) {
    fetchData();
    return;
  }

  const result = await storage.value.find({
    selector: {
      'biblio.games.0.title': { $regex: new RegExp(searchTitle.value, 'i') }
    }
  });

  gamesData.value = result.docs as Game[];
};

// Initialisation DB + réplication officielle
const initDatabase = () => {
  const localDB = new PouchDB('infradon-local');
  storage.value = localDB;

  const remoteDB = new PouchDB('http://admin:admin@localhost:5984/infradon2');

  // Réplication initiale (toujours)
  localDB
    .replicate.from(remoteDB)
    .on('complete', () => fetchData());

  // LIVE sync désactivée volontairement (mode manuel)
  // activée uniquement via un bouton

  // Index
  localDB.createIndex({
    index: { fields: ['biblio.games.0.title'] }
  });
};

// Récupérer tous les documents
const fetchData = async () => {
  if (!storage.value) return;

  const result = await storage.value.allDocs({ include_docs: true });
  gamesData.value = result.rows
    .map(r => r.doc as Game)
    .filter(doc => doc?.biblio?.games)
    .filter((doc) => !doc._id.startsWith("_"));
};

// Ajouter un jeu
const addGame = async (title: string, editor: string, country?: string, release?: number) => {
  if (!storage.value) return;

  const doc: Game = {
    _id: `game_${Date.now()}`,
    biblio: {
      games: [
        { title, editor, country, release: release || new Date().getFullYear() }
      ]
    }
  };

  await storage.value.put(doc);
  await fetchData();
};

// Modifier un jeu
const updateGame = async (id: string, title: string, editor: string, country?: string, release?: number) => {
  if (!storage.value) return;

  const doc = await storage.value.get<Game>(id);

  doc.biblio.games[0] = {
    title,
    editor,
    country,
    release: release || new Date().getFullYear(),
  };

  await storage.value.put(doc);
  editingId.value = null;

  newGame.value = {
    title: '',
    editor: '',
    country: '',
    release: new Date().getFullYear(),
  };

  await fetchData();
};

// Supprimer un jeu
const deleteGame = async (id: string) => {
  if (!storage.value) return;

  const doc = await storage.value.get(id);
  await storage.value.remove(doc);
  await fetchData();
};

// Soumission formulaire
const submitNewGame = () => {
  if (editingId.value) {
    updateGame(
      editingId.value,
      newGame.value.title,
      newGame.value.editor,
      newGame.value.country,
      newGame.value.release
    );
  } else {
    addGame(
      newGame.value.title,
      newGame.value.editor,
      newGame.value.country,
      newGame.value.release
    );
  }
};

// Remplissage formulaire
const editGame = (game: Game) => {
  editingId.value = game._id;
  newGame.value = {
    title: game.biblio.games[0].title,
    editor: game.biblio.games[0].editor,
    country: game.biblio.games[0].country || '',
    release: game.biblio.games[0].release,
  };
};

// synchronisation manuelle
const watchDistantChanges = async () => {
  if (!storage.value || isOffline.value) return;

  const localDB = storage.value;
  const remoteDB = new PouchDB('http://admin:admin@localhost:5984/infradon2');

  await localDB.replicate.to(remoteDB);
  await localDB.replicate.from(remoteDB);

  await fetchData();
};

onMounted(() => {
  initDatabase();
});
</script>

<template>
  <h1>Games List</h1>

  <!-- Mode offline -->
  <!-- Section Sync + Offline alignés -->
  <div class="sync-offline-row">
    <label class="toggle-label">
      <input type="checkbox" v-model="isOffline" />
      <span class="toggle-custom"></span>
      Mode Offline
    </label>

    <button class="sync-btn" @click="watchDistantChanges" :disabled="isOffline">
      Synchronisation
    </button>
  </div>


  <!-- Barre de recherche -->
  <input v-model="searchTitle" @input="searchGames" placeholder="Rechercher un jeu par titre..."
    style="margin-bottom: 15px; padding: 5px" />

  <form @submit.prevent="submitNewGame" class="game-form">
    <h2>{{ editingId ? 'Modifier un jeu' : 'Ajouter un jeu' }}</h2>

    <div>
      <label for="title">Titre :</label>
      <input id="title" v-model="newGame.title" required />
    </div>

    <div>
      <label for="editor">Éditeur :</label>
      <input id="editor" v-model="newGame.editor" required />
    </div>

    <div>
      <label for="country">Pays :</label>
      <input id="country" v-model="newGame.country" />
    </div>

    <div>
      <label for="release">Année de sortie :</label>
      <input id="release" v-model.number="newGame.release" type="number" required />
    </div>

    <button type="submit">
      {{ editingId ? 'Mettre à jour' : 'Ajouter le jeu' }}
    </button>

    <button v-if="editingId" type="button"
      @click="editingId = null; newGame = { title: '', editor: '', country: '', release: new Date().getFullYear() }">
      Annuler
    </button>
  </form>

  <div v-if="gamesData.length > 0">
    <h2>Liste des jeux</h2>

    <div v-for="game in gamesData" :key="game._id" class="game-card">
      <div v-for="(g, index) in game.biblio.games" :key="index">
        <h3>{{ g.title }}</h3>
        <p><strong>Éditeur :</strong> {{ g.editor }}</p>
        <p v-if="g.country"><strong>Pays :</strong> {{ g.country }}</p>
        <p><strong>Année :</strong> {{ g.release }}</p>

        <button @click="editGame(game)">Modifier</button>
        <button @click="deleteGame(game._id)">Supprimer</button>
      </div>
    </div>
  </div>

  <p v-else>Aucun jeu trouvé.</p>
</template>
