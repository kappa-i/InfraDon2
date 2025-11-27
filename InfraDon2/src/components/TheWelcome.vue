<script setup lang="ts">
import { ref, onMounted } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find';

PouchDB.plugin(PouchDBFind);

interface Comment {
  _id?: string;
  _rev?: string;
  post_id: string;
  text: string;
  created_at: string;
}

interface Post {
  _id?: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  likes: number;
}

const postsDB = ref();
const commentsDB = ref();
const postsData = ref<Post[]>([]);
const commentsData = ref<{ [key: string]: Comment[] }>({});
const isOffline = ref(false);
const searchQuery = ref("");

const postsSyncHandler = ref<any>(null);
const commentsSyncHandler = ref<any>(null);

const remotePostsDB = ref<any>(null);
const remoteCommentsDB = ref<any>(null);

const REMOTE_POSTS = "http://admin:root@localhost:5984/dbinfradon2gab_posts";
const REMOTE_COMMENTS = "http://admin:root@localhost:5984/dbinfradon2gab_comments";

const initRemoteDBs = () => {
  if (!remotePostsDB.value) {
    remotePostsDB.value = new PouchDB(REMOTE_POSTS);
  }
  if (!remoteCommentsDB.value) {
    remoteCommentsDB.value = new PouchDB(REMOTE_COMMENTS);
  }
};

const toggleOffline = () => {
  isOffline.value = !isOffline.value;

  if (isOffline.value) {
    
    postsSyncHandler.value?.cancel();
    commentsSyncHandler.value?.cancel();

    postsSyncHandler.value = null;
    commentsSyncHandler.value = null;

    console.log("🔴 OFFLINE : Synchronisation coupée");
  } else {
    console.log("🟢 ONLINE : Tentative de reconnexion...");
    initRemoteDBs(); 
    syncToServer();
  }
};

const syncToServer = async () => {

  if (isOffline.value || !remotePostsDB.value || !remoteCommentsDB.value) {
    console.warn("Impossible de synchroniser : mode offline ou bases distantes manquantes");
    return;
  }

  try {
    console.log("⏳ Synchronisation ponctuelle en cours...");
    
    await postsDB.value.replicate.to(remotePostsDB.value);
    await commentsDB.value.replicate.to(remoteCommentsDB.value);
    
    await postsDB.value.replicate.from(remotePostsDB.value);
    await commentsDB.value.replicate.from(remoteCommentsDB.value);
    
    console.log("✅ Synchronisation ponctuelle terminée");
    
    startContinuousSync();
    
    fetchData();
  } catch (err) {
    console.error("❌ Erreur de synchronisation:", err);
  }
};

const startContinuousSync = () => {
  if (isOffline.value) return;

  postsSyncHandler.value?.cancel();
  commentsSyncHandler.value?.cancel();

  postsSyncHandler.value = postsDB.value.sync(remotePostsDB.value, {
    live: true,
    retry: true
  }).on('change', () => {

    fetchData(); 
  }).on('error', (err: any) => {
      console.error("Erreur Sync Live:", err);
  });

  commentsSyncHandler.value = commentsDB.value.sync(remoteCommentsDB.value, {
    live: true,
    retry: true
  }).on('change', () => {
      fetchComments();
  });
  
};

const initDatabase = async () => {
  postsDB.value = new PouchDB("Posts");
  commentsDB.value = new PouchDB("Comments");

  await postsDB.value.createIndex({ index: { fields: ['likes'] } });
  await commentsDB.value.createIndex({
    index: { fields: ['post_id', 'created_at'] }
  });

  if (!isOffline.value) {
    initRemoteDBs();
    startContinuousSync(); 
  }

  fetchData();
};

const runFactory = async () => {
  const dummyPosts = Array.from({ length: 10 }, (_, i) => ({
    post_name: `Post Factory ${i}`,
    post_content: `Contenu généré ${Math.random()}`,
    likes: Math.floor(Math.random() * 100)
  }));
  await postsDB.value.bulkDocs(dummyPosts);
  fetchData();
};

const searchAndSort = async () => {
  if (!searchQuery.value) {
    fetchPosts();
    return;
  }
  const result = await postsDB.value.allDocs({ include_docs: true });
  postsData.value = result.rows
    .map((row: any) => row.doc)
    .filter((doc: Post) => doc.post_name?.toLowerCase().includes(searchQuery.value.toLowerCase()))
    .sort((a: Post, b: Post) => b.likes - a.likes);
  fetchComments();
};

const addPost = async () => {
  await postsDB.value.post({
    post_name: 'Chillout',
    post_content: 'Je suis un gars trop chill',
    likes: 0
  });
  fetchData();
};

const deletePost = async (post: Post) => {
  await postsDB.value.remove(post);
  const comments = await commentsDB.value.find({ selector: { post_id: post._id } });
  const deleteDocs = comments.docs.map((doc: any) => ({ ...doc, _deleted: true }));
  await commentsDB.value.bulkDocs(deleteDocs);
  fetchData();
};

const updatePost = async (post: Post) => {
  const doc = await postsDB.value.get(post._id);
  doc.post_name = "Modifié !";
  doc.post_content = new Date().toISOString();
  await postsDB.value.put(doc);
  fetchData();
};

const likePost = async (post: Post) => {
  const doc = await postsDB.value.get(post._id);
  doc.likes = (doc.likes || 0) + 1;
  await postsDB.value.put(doc);
  fetchData();
};

const sortByLikes = async () => {
  const result = await postsDB.value.find({
    selector: { likes: { $gte: 0 } },
    sort: [{ 'likes': 'desc' }]
  });
  postsData.value = result.docs;
};

const addComment = async (post: Post) => {
  await commentsDB.value.post({
    post_id: post._id!,
    text: `Commentaire ${(Math.random()*1000).toFixed()}`,
    created_at: new Date().toISOString()
  });
  fetchComments();
};

const deleteComment = async (comment: Comment) => {
  await commentsDB.value.remove(comment);
  fetchComments();
};

const fetchPosts = async () => {
  const result = await postsDB.value.find({
    selector: { likes: { $gte: 0 } },
    sort: [{ 'likes': 'desc' }]
  });
  postsData.value = result.docs;
};

const fetchComments = async () => {
  const result = await commentsDB.value.allDocs({ include_docs: true });
  commentsData.value = result.rows.reduce((acc: any, row: any) => {
    const comment = row.doc as Comment;
    if (comment.post_id) {
      if (!acc[comment.post_id]) acc[comment.post_id] = [];
      acc[comment.post_id].push(comment);
    }
    return acc;
  }, {});
};

const fetchData = () => {
  if (searchQuery.value) {
    searchAndSort();
  } else {
    fetchPosts();
    fetchComments();
  }
};

onMounted(initDatabase);
</script>

<template>
  <div>
    <h1 style="color: white;">InfraDon2</h1>

    <div>
      <button @click="toggleOffline" :style="{ 
        backgroundColor: isOffline ? '#fecaca' : '#bbf7d0',
        fontWeight: 'bold'
      }">
        {{ isOffline ? "🔴 OFFLINE" : "🟢 ONLINE" }}
      </button>
      <button @click="runFactory">Lancer Factory</button>
      <button @click="sortByLikes">Trier par likes</button>
    </div>

    <hr />

    <div>
      <input v-model="searchQuery" placeholder="Rechercher un post..." />
      <button @click="searchAndSort">Rechercher & Trier</button>
    </div>

    <button @click="addPost">Ajouter un post</button>

    <h2 style="color: white;">Posts ({{ postsData.length }})</h2>
    <article v-for="post in postsData" :key="post._id">
      <h3>{{ post.post_name }} (Likes: {{ post.likes }})</h3>
      <p>{{ post.post_content }}</p>

      <button @click="likePost(post)">Like</button>
      <button @click="updatePost(post)">Modifier</button>
      <button @click="deletePost(post)">Supprimer</button>
      <button @click="addComment(post)">Ajouter Commentaire</button>

      <h4>Commentaires:</h4>

      <div v-for="comment in commentsData[post._id!]" :key="comment._id">
        <p>{{ comment.text }}</p>
        <button @click="deleteComment(comment)">Supprimer Commentaire</button>
      </div>

    </article>
  </div>
</template>

<style scoped>

div {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  color: #374151;
  line-height: 1.6;
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
}

h1,
h2,
h3,
h4 {
  color: #111827;
  font-weight: 600;
  margin-bottom: 1rem;
}

h1 {
  font-size: 2rem;
  text-align: center;
  margin-bottom: 2rem;
}

h2 {
  font-size: 1.5rem;
  margin-top: 2rem;
  border-bottom: 2px solid #f3f4f6;
  padding-bottom: 0.5rem;
}

h3 {
  font-size: 1.25rem;
  margin: 0;
}

hr {
  border: 0;
  border-top: 1px solid #e5e7eb;
  margin: 2rem 0;
}

/* --- Inputs & Boutons --- */
input {
  padding: 0.6rem 1rem;
  border: 1px solid #d1d5db;
  border-radius: 6px;
  margin-right: 0.5rem;
  font-size: 0.95rem;
  width: 250px;
  transition: border-color 0.2s;
}

input:focus {
  outline: none;
  border-color: #3b82f6;
}

button {
  background-color: white;
  border: 1px solid #d1d5db;
  padding: 0.5rem 1rem;
  border-radius: 6px;
  cursor: pointer;
  font-size: 0.9rem;
  font-weight: 500;
  color: #374151;
  transition: all 0.2s ease;
  margin-right: 0.5rem;
  margin-bottom: 0.5rem;
}

button:hover {
  background-color: #f9fafb;
  border-color: #9ca3af;
  color: #111827;
}

button:active {
  transform: translateY(1px);
}

/* --- Cartes des Posts --- */
article {
  background: white;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
  transition: transform 0.2s, box-shadow 0.2s;
}

article:hover {
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
}

article p {
  margin: 0.5rem 0 1.5rem 0;
  color: #4b5563;
}

/* --- Section Commentaires --- */
h4 {
  font-size: 1rem;
  margin-top: 1.5rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #9ca3af;
}

article>div {
  background-color: #f9fafb;
  border-radius: 6px;
  padding: 1rem;
  margin-top: 1rem;
}

article>div>div {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #e5e7eb;
  padding: 0.5rem 0;
}

article>div>div:last-child {
  border-bottom: none;
}

article>div p {
  margin: 0;
  font-size: 0.9rem;
}

div>div {
  margin-bottom: 1rem;
}
</style>