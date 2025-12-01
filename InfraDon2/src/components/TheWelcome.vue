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

interface CommentDoc {
  _id: string;
  _rev?: string;
  type: 'comment';
  gameId: string;
  text: string;
  author?: string;
  createdAt: number;
}

interface LikeDoc {
  _id: string;
  _rev?: string;
  type: 'like';
  gameId: string;
  createdAt: number;
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

// comments & likes state
const commentsByGame = ref<Record<string, CommentDoc[]>>({});
const showAllComments = ref<Record<string, boolean>>({});
const newCommentText = ref<Record<string, string>>({});
const likesCount = ref<Record<string, number>>({});
const likesDocsByGame = ref<Record<string, LikeDoc[]>>({});

// helper
const getRemoteDB = () => new PouchDB('http://admin:admin@localhost:5984/infradon2');

// search games by title
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
  for (const g of gamesData.value) {
    fetchCommentsForGame(g._id);
    fetchLikesForGame(g._id);
  }
};

// Initialisation DB + réplication officielle
const initDatabase = () => {
  const localDB = new PouchDB('infradon-local');
  storage.value = localDB;

  const remoteDB = new PouchDB('http://admin:admin@localhost:5984/infradon2');

  // Réplication initiale
  localDB
    .replicate.from(remoteDB)
    .on('complete', () => fetchData());
};

// Récupérer tous les documents
const fetchData = async () => {
  if (!storage.value) return;

  const result = await storage.value.find({
    selector: { _id: { $gt: null } }
  });

  gamesData.value = result.docs
    .filter((doc: any) => doc?.biblio?.games)
    .filter((doc: any) => !doc._id.startsWith("_"));

  for (const g of gamesData.value) {
    await fetchCommentsForGame(g._id);
    await fetchLikesForGame(g._id);
  }
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

  doc.biblio.games[0] = { title, editor, country, release: release || new Date().getFullYear() };

  await storage.value.put(doc);
  editingId.value = null;

  newGame.value = { title: '', editor: '', country: '', release: new Date().getFullYear() };

  await fetchData();
};

// Supprimer un jeu
const deleteGame = async (id: string) => {
  if (!storage.value) return;

  const doc = await storage.value.get(id);
  await storage.value.remove(doc);

  if (storage.value) {
    const cRes = await storage.value.find({ selector: { type: 'comment', gameId: id } });
    for (const c of cRes.docs) await storage.value.remove(c);

    const lRes = await storage.value.find({ selector: { type: 'like', gameId: id } });
    for (const l of lRes.docs) await storage.value.remove(l);
  }
  await fetchData();
};

// Soumission formulaire
const submitNewGame = () => {
  if (editingId.value) {
    updateGame(editingId.value, newGame.value.title, newGame.value.editor, newGame.value.country, newGame.value.release);
  } else {
    addGame(newGame.value.title, newGame.value.editor, newGame.value.country, newGame.value.release);
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

// Comments functions
const addComment = async (gameId: string) => {
  if (!storage.value) return;
  const text = (newCommentText.value[gameId] || '').trim();
  if (!text) return;

  const doc: CommentDoc = { _id: `comment_${Date.now()}`, type: 'comment', gameId, text, createdAt: Date.now() };
  await storage.value.put(doc);
  newCommentText.value[gameId] = '';
  await fetchCommentsForGame(gameId);
};

const fetchCommentsForGame = async (gameId: string) => {
  if (!storage.value) return;
  const res = await storage.value.find({ selector: { type: 'comment', gameId } });
  commentsByGame.value[gameId] = (res.docs as CommentDoc[]).sort((a, b) => b.createdAt - a.createdAt);
  if (showAllComments.value[gameId] === undefined) showAllComments.value[gameId] = false;
};

const deleteComment = async (commentId: string, gameId?: string) => {
  if (!storage.value) return;
  const doc = await storage.value.get(commentId);
  await storage.value.remove(doc);
  if (gameId) await fetchCommentsForGame(gameId);
};

// Likes functions
const fetchLikesForGame = async (gameId: string) => {
  if (!storage.value) return;
  const res = await storage.value.find({ selector: { type: 'like', gameId } });
  likesDocsByGame.value[gameId] = res.docs as LikeDoc[];
  likesCount.value[gameId] = likesDocsByGame.value[gameId]?.length || 0;
};

const toggleLike = async (gameId: string) => {
  if (!storage.value) return;

  const likes = likesDocsByGame.value[gameId] || [];
  if (likes.length > 0) {
    // unlike the most recent
    const doc = await storage.value.get(likes[likes.length - 1]._id);
    await storage.value.remove(doc);
  } else {
    // add a new like
    const doc: LikeDoc = { _id: `like_${Date.now()}`, type: 'like', gameId, createdAt: Date.now() };
    await storage.value.put(doc);
  }
  await fetchLikesForGame(gameId);
};

// synchronisation manuelle
const watchDistantChanges = async () => {
  if (!storage.value || isOffline.value) return;

  const localDB = storage.value;
  const remoteDB = getRemoteDB();

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

    <button type="submit">{{ editingId ? 'Mettre à jour' : 'Ajouter le jeu' }}</button>

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

        <!-- Like bouton cœur -->
        <div style="margin-top:6px;">
          <button @click="() => toggleLike(game._id)"
            style="background:none; border:none; cursor:pointer; font-size:1.3em;">
            ❤️
          </button>
          <span>{{ likesCount[game._id] || 0 }}</span>
        </div>

        <!-- Commentaire -->
        <div style="margin-top:8px;">
          <input v-model="newCommentText[game._id]" placeholder="Ajouter un commentaire..."
            style="width:80%; padding:6px; margin-right:6px;" />
          <button @click="() => addComment(game._id)">Ajouter</button>
        </div>

        <ul v-if="commentsByGame[game._id] && commentsByGame[game._id].length" style="margin-top:8px;">
          <li v-for="c in commentsByGame[game._id]" :key="c._id"
            style="margin-bottom:4px; display:flex; align-items:center; justify-content:space-between;">
            <span>{{ c.text }}</span>
            <button @click="() => deleteComment(c._id, game._id)"
              style="background:none; border:none; color:red; cursor:pointer;">×</button>
          </li>
        </ul>

        <div style="margin-top:10px;">
          <button @click="editGame(game)">Modifier</button>
          <button @click="deleteGame(game._id)">Supprimer</button>
        </div>
      </div>
    </div>
  </div>

  <p v-else>Aucun jeu trouvé.</p>
</template>