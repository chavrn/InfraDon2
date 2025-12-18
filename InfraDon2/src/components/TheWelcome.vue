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
  _attachments?: {
    [key: string]: {
      content_type: string;
      data: string | Blob;
    };
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
const gamesDB = ref<PouchDB.Database>();
const commentsDB = ref<PouchDB.Database>();
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

// Tri ⭐ NOUVEAU
const sortBy = ref<'default' | 'likes'>('default');

// comments & likes state
const commentsByGame = ref<Record<string, CommentDoc[]>>({});
const showAllComments = ref<Record<string, boolean>>({});
const newCommentText = ref<Record<string, string>>({});
const likesCount = ref<Record<string, number>>({});
const likesDocsByGame = ref<Record<string, LikeDoc[]>>({});

// Images
const selectedImage = ref<File | null>(null);
const imagePreview = ref<string>('');
const gameImages = ref<Record<string, string>>({});

// Gestion sélection image
const onImageSelect = (event: Event) => {
  const input = event.target as HTMLInputElement;
  if (input.files && input.files[0]) {
    selectedImage.value = input.files[0];
    const reader = new FileReader();
    reader.onload = (e) => {
      imagePreview.value = e.target?.result as string;
    };
    reader.readAsDataURL(input.files[0]);
  }
};

// Charger image pour un jeu
const loadGameImage = async (gameId: string) => {
  if (!gamesDB.value) return;
  try {
    const doc = await gamesDB.value.get(gameId, { attachments: true, binary: false });
    if (doc._attachments && doc._attachments['image']) {
      const attachment = doc._attachments['image'];
      if (typeof attachment.data === 'string') {
        gameImages.value[gameId] = `data:${attachment.content_type};base64,${attachment.data}`;
      }
    }
  } catch (err) {
    console.error('Error loading image:', err);
  }
};

// Supprimer l'image d'un jeu
const deleteGameImage = async (gameId: string) => {
  if (!gamesDB.value) return;
  try {
    const doc = await gamesDB.value.get(gameId);
    if (doc._attachments && doc._attachments['image']) {
      await gamesDB.value.removeAttachment(gameId, 'image', doc._rev);
      delete gameImages.value[gameId];
      await fetchData();
    }
  } catch (err) {
    console.error('Error deleting image:', err);
  }
};

// search games by title
const searchGames = async () => {
  if (!gamesDB.value) return;

  if (!searchTitle.value.trim()) {
    fetchData();
    return;
  }

  const result = await gamesDB.value.find({
    selector: {
      'biblio.games.0.title': { $regex: new RegExp(searchTitle.value, 'i') }
    }
  });

  gamesData.value = result.docs as Game[];
  for (const g of gamesData.value) {
    fetchCommentsForGame(g._id);
    fetchLikesForGame(g._id);
    loadGameImage(g._id);
  }
};

// Fonction pour trier les jeux ⭐ NOUVEAU
const getSortedGames = () => {
  if (sortBy.value === 'likes') {
    return [...gamesData.value].sort((a, b) => {
      const likesA = likesCount.value[a._id] || 0;
      const likesB = likesCount.value[b._id] || 0;
      return likesB - likesA; // Du plus au moins
    });
  }
  return gamesData.value;
};

// Toggle du tri ⭐ NOUVEAU
const toggleSort = () => {
  sortBy.value = sortBy.value === 'default' ? 'likes' : 'default';
};

// Initialisation DB + réplication officielle
const initDatabase = async () => {
  const localGamesDB = new PouchDB('infradon-games-local');
  gamesDB.value = localGamesDB;

  const localCommentsDB = new PouchDB('infradon-comments-local');
  commentsDB.value = localCommentsDB;

  // Créer index pour le tri des commentaires
  try {
    await localCommentsDB.createIndex({
      index: { fields: ['type', 'gameId', 'createdAt'] }
    });
  } catch (err) {
    console.error('Index creation error:', err);
  }

  const remoteGamesDB = new PouchDB('http://admin:admin@localhost:5984/macollection_vernay-charline');
  const remoteCommentsDB = new PouchDB('http://admin:admin@localhost:5984/macollection-comments_vernay-charline');

  // Réplication initiale
  localGamesDB
    .replicate.from(remoteGamesDB)
    .on('complete', () => fetchData());

  localCommentsDB.replicate.from(remoteCommentsDB);
};

// Récupérer tous les documents
const fetchData = async () => {
  if (!gamesDB.value) return;

  const result = await gamesDB.value.find({
    selector: { _id: { $gt: null } }
  });

  gamesData.value = result.docs
    .filter((doc: any) => doc?.biblio?.games)
    .filter((doc: any) => !doc._id.startsWith("_"));

  for (const g of gamesData.value) {
    await fetchCommentsForGame(g._id);
    await fetchLikesForGame(g._id);
    await loadGameImage(g._id);
  }
};

// Ajouter un jeu
const addGame = async (title: string, editor: string, country?: string, release?: number) => {
  if (!gamesDB.value) return;

  const doc: Game = {
    _id: `game_${Date.now()}`,
    biblio: {
      games: [
        { title, editor, country, release: release || new Date().getFullYear() }
      ]
    }
  };

  const savedDoc = await gamesDB.value.put(doc);

  if (selectedImage.value) {
    const imageFile = selectedImage.value;
    const imageType = selectedImage.value.type;

    const reader = new FileReader();
    reader.onload = async (e) => {
      if (e.target && e.target.result) {
        const base64String = e.target.result as string;
        const base64Data = base64String.split(',')[1];
        await gamesDB.value!.putAttachment(
          doc._id,
          'image',
          savedDoc.rev,
          base64Data,
          imageType
        );
        await fetchData();
      }
    };
    reader.readAsDataURL(imageFile);
  } else {
    await fetchData();
  }

  newGame.value = { title: '', editor: '', country: '', release: new Date().getFullYear() };
  selectedImage.value = null;
  imagePreview.value = '';
};

// Modifier un jeu
const updateGame = async (id: string, title: string, editor: string, country?: string, release?: number) => {
  if (!gamesDB.value) return;

  const doc = await gamesDB.value.get<Game>(id);

  doc.biblio.games[0] = { title, editor, country, release: release || new Date().getFullYear() };

  const savedDoc = await gamesDB.value.put(doc);

  if (selectedImage.value) {
    const imageFile = selectedImage.value;
    const imageType = selectedImage.value.type;

    const reader = new FileReader();
    reader.onload = async (e) => {
      if (e.target && e.target.result) {
        const base64String = e.target.result as string;
        const base64Data = base64String.split(',')[1];
        await gamesDB.value!.putAttachment(
          id,
          'image',
          savedDoc.rev,
          base64Data,
          imageType
        );
        await fetchData();
      }
    };
    reader.readAsDataURL(imageFile);
  } else {
    await fetchData();
  }

  editingId.value = null;
  newGame.value = { title: '', editor: '', country: '', release: new Date().getFullYear() };
  selectedImage.value = null;
  imagePreview.value = '';
};

// Supprimer un jeu
const deleteGame = async (id: string) => {
  if (!gamesDB.value || !commentsDB.value) return;

  const doc = await gamesDB.value.get(id);
  await gamesDB.value.remove(doc);

  const cRes = await commentsDB.value.find({ selector: { type: 'comment', gameId: id } });
  for (const c of cRes.docs) await commentsDB.value.remove(c);

  const lRes = await commentsDB.value.find({ selector: { type: 'like', gameId: id } });
  for (const l of lRes.docs) await commentsDB.value.remove(l);

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
const editGame = async (game: Game) => {
  editingId.value = game._id;
  newGame.value = {
    title: game.biblio.games[0].title,
    editor: game.biblio.games[0].editor,
    country: game.biblio.games[0].country || '',
    release: game.biblio.games[0].release,
  };

  if (gameImages.value[game._id]) {
    imagePreview.value = gameImages.value[game._id];
  }
};

// Comments functions
const addComment = async (gameId: string) => {
  if (!commentsDB.value) return;
  const text = (newCommentText.value[gameId] || '').trim();
  if (!text) return;

  const doc: CommentDoc = { _id: `comment_${Date.now()}`, type: 'comment', gameId, text, createdAt: Date.now() };
  const result = await commentsDB.value.put(doc);

  const newComment: CommentDoc = { ...doc, _rev: result.rev };
  if (!commentsByGame.value[gameId]) commentsByGame.value[gameId] = [];
  commentsByGame.value[gameId].unshift(newComment);

  newCommentText.value[gameId] = '';
};

const fetchCommentsForGame = async (gameId: string) => {
  if (!commentsDB.value) return;
  const res = await commentsDB.value.find({
    selector: { type: 'comment', gameId },
    sort: [{ type: 'asc' }, { gameId: 'asc' }, { createdAt: 'desc' }]
  });
  commentsByGame.value[gameId] = res.docs as CommentDoc[];
  if (showAllComments.value[gameId] === undefined) showAllComments.value[gameId] = false;
};

const deleteComment = async (commentId: string, gameId?: string) => {
  if (!commentsDB.value) return;
  const doc = await commentsDB.value.get(commentId);
  await commentsDB.value.remove(doc);
  if (gameId) await fetchCommentsForGame(gameId);
};

// Likes functions
const fetchLikesForGame = async (gameId: string) => {
  if (!commentsDB.value) return;
  const res = await commentsDB.value.find({ selector: { type: 'like', gameId } });
  likesDocsByGame.value[gameId] = res.docs as LikeDoc[];
  likesCount.value[gameId] = likesDocsByGame.value[gameId]?.length || 0;
};

const toggleLike = async (gameId: string) => {
  if (!commentsDB.value) return;

  const doc: LikeDoc = { _id: `like_${Date.now()}`, type: 'like', gameId, createdAt: Date.now() };
  await commentsDB.value.put(doc);

  if (!likesCount.value[gameId]) likesCount.value[gameId] = 0;
  likesCount.value[gameId]++;
};

// synchronisation manuelle
const watchDistantChanges = async () => {
  if (!gamesDB.value || !commentsDB.value || isOffline.value) return;

  const remoteGamesDB = new PouchDB('http://admin:admin@localhost:5984/macollection_vernay-charline');
  const remoteCommentsDB = new PouchDB('http://admin:admin@localhost:5984/macollection-comments_vernay-charline');

  await gamesDB.value.replicate.to(remoteGamesDB);
  await gamesDB.value.replicate.from(remoteGamesDB);

  await commentsDB.value.replicate.to(remoteCommentsDB);
  await commentsDB.value.replicate.from(remoteCommentsDB);

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

    <div>
      <label for="image">Image :</label>
      <input id="image" type="file" accept="image/*" @change="onImageSelect" />
      <img v-if="imagePreview" :src="imagePreview" alt="Preview"
        style="max-width: 200px; margin-top: 10px; border-radius: 4px;" />
    </div>

    <button type="submit">{{ editingId ? 'Mettre à jour' : 'Ajouter le jeu' }}</button>

    <button v-if="editingId" type="button"
      @click="editingId = null; newGame = { title: '', editor: '', country: '', release: new Date().getFullYear() }; selectedImage = null; imagePreview = '';">
      Annuler
    </button>
  </form>

  <!-- Bouton tri ⭐ NOUVEAU -->
  <div v-if="gamesData.length > 0" style="margin-bottom: 20px; display: flex; gap: 12px; align-items: center;">
    <button @click="toggleSort" :style="{
      background: sortBy === 'likes' ? 'var(--accent)' : 'var(--fg)',
      borderColor: sortBy === 'likes' ? 'var(--accent)' : 'var(--fg)'
    }">
      {{ sortBy === 'likes' ? '❤️ Trier par likes (actif)' : 'Trier par likes' }}
    </button>
  </div>

  <div v-if="gamesData.length > 0">
    <h2>Liste des jeux</h2>

    <!-- ⭐ CHANGÉ: getSortedGames() au lieu de gamesData -->
    <div v-for="game in getSortedGames()" :key="game._id" class="game-card">
      <div v-for="(g, index) in game.biblio.games" :key="index"
        style="display: flex; align-items: center; width: 100%;">

        <div v-if="gameImages[game._id]" style="position: relative; margin-right: 20px;">

          <img :src="gameImages[game._id]" alt="Game image"
            style="width: 120px; height: 120px; object-fit: cover; border-radius: 8px;" />

          <button @click="() => deleteGameImage(game._id)" class="delete-image-btn">
            ×
          </button>

        </div>

        <div style="flex: 1;">
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
  </div>

  <p v-else>Aucun jeu trouvé.</p>
</template>