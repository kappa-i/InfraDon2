<script setup lang="ts">
import { ref, onMounted } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find'; 

PouchDB.plugin(PouchDBFind);

declare interface Comment {
  _id?: string;
  text: string;
}

declare interface Post {
  _id?: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  likes: number;
  comments?: Comment[];
}

const storage = ref();
const postsData = ref<Post[]>([]);
const isOffline = ref(false);
const searchQuery = ref("");
const syncHandler = ref<any>(null);

const toggleOffline = () => {
  isOffline.value = !isOffline.value;

  if (isOffline.value) {
    console.log("Mode OFFLINE activé");

    if (syncHandler.value) {
      syncHandler.value.cancel();
      syncHandler.value = null;
    }
  } else {
    console.log("Mode ONLINE activé");
    startContinuousSync();  
  }
};


const initDatabase = () => {
  const db = new PouchDB("Posts");
  if (db) {
    storage.value = db;
    
    db.createIndex({
      index: { fields: ['likes', 'post_name'] }
    }).then(() => console.log("Index créé"));

    db.replicate.from("http://admin:root@localhost:5984/dbinfradon2gab").then(() => {
      fetchData();
      startContinuousSync();
    }).catch((err) => console.log(err));
  }
}

const startContinuousSync = () => {

  if (isOffline.value) {
    console.log("Sync annulée (offline)");
    return;
  }

  const remoteDB = new PouchDB('http://admin:root@localhost:5984/dbinfradon2gab');

  if (syncHandler.value) {
    syncHandler.value.cancel();
  }

  syncHandler.value = storage.value.sync(remoteDB, {
    live: true,
    retry: true
  })
};


const runFactory = async () => {
  const dummyPosts = [];
  for (let i = 0; i < 10; i++) {
    dummyPosts.push({
      post_name: `Post Factory ${i}`,
      post_content: `Contenu généré ${Math.random()}`,
      likes: Math.floor(Math.random() * 100),
      comments: []
    });
  }
  await storage.value.bulkDocs(dummyPosts);
  fetchData();
};

const searchAndSort = () => {
  const selector: any = { likes: { $gte: 0 } };
  
  if (searchQuery.value) {
    storage.value.allDocs({ include_docs: true }).then((result: any) => {
      let filtered = result.rows.map((row: any) => row.doc);
      filtered = filtered.filter((doc: Post) => 
        doc.post_name && doc.post_name.toLowerCase().includes(searchQuery.value.toLowerCase())
      );
      filtered.sort((a: Post, b: Post) => (b.likes || 0) - (a.likes || 0));
      postsData.value = filtered;
    });
  } else {
    storage.value.find({
      selector: selector,
      sort: [{ 'likes': 'desc' }] 
    }).then((result: any) => {
      postsData.value = result.docs;
    }).catch((err: any) => console.error(err));
  }
};

const addPost = () => {
  const newPost: Post = {
    post_name: 'Chillout',
    post_content: 'Je suis un gars trop chill',
    likes: 0,
    comments: []
  };
  storage.value.post(newPost).then(() => fetchData());
};

const deletePost = (post: Post) => {
  storage.value.remove(post).then(() => fetchData());
};

const updatePost = (post: Post) => {
  storage.value.get(post._id).then((doc: any) => {
    doc.post_name = "Modifié !";
    doc.post_content = new Date().toISOString();
    return storage.value.put(doc);
  }).then(() => fetchData());
};

const likePost = (post: Post) => {
  storage.value.get(post._id).then((doc: any) => {
    doc.likes = (doc.likes || 0) + 1;
    return storage.value.put(doc);
  }).then(() => fetchData());
};

const sortByLikes = () => {
  postsData.value.sort((a, b) => (b.likes || 0) - (a.likes || 0));
};


const addComment = (post: Post) => {
  storage.value.get(post._id).then((doc: any) => {
    if (!doc.comments) {
      doc.comments = [];
    }
    doc.comments.push({
      _id: `comment_${Date.now()}`,
      text: `Commentaire ${Math.random().toFixed(2)}`
    });
    return storage.value.put(doc);
  }).then(() => fetchData());
};

const deleteComment = (post: Post, commentId: string) => {
  storage.value.get(post._id).then((doc: any) => {
    doc.comments = doc.comments.filter((c: Comment) => c._id !== commentId);
    return storage.value.put(doc);
  }).then(() => fetchData());
};

const fetchData = () => {
  if(searchQuery.value === "") {
      storage.value.allDocs({ include_docs: true }).then((result: any) => {
      postsData.value = result.rows.map((row: any) => row.doc);
      
    });
  } else {
    searchAndSort();
  }
};

onMounted(() => {
  initDatabase();
  fetchData();
});
</script>

<template>
  <div>
    <h1 style="color: white;">InfraDon2</h1>

    <div>
      <button @click="toggleOffline">{{ isOffline ? "Passer Online" : "Passer Offline" }}</button>
      <button @click="runFactory">Lancer Factory</button>
      <button @click="sortByLikes">Trier par likes</button>

    </div>

    <hr/>

    <div>
      <input v-model="searchQuery" placeholder="Rechercher un post..." />
      <button @click="searchAndSort">Rechercher & Trier</button>
    </div>

    <button @click="addPost">Ajouter un post</button>

    <h2>Posts</h2>
    <article v-for="post in postsData" :key="post._id">
      <h3>{{ post.post_name }} (Likes: {{ post.likes }})</h3>
      <p>{{ post.post_content }}</p>
      
      <button @click="likePost(post)">Like</button>
      <button @click="updatePost(post)">Modifier</button>
      <button @click="deletePost(post)">Supprimer</button>
      <button @click="addComment(post)">Ajouter Commentaire</button>

      
      <div v-if="post.comments && post.comments.length > 0">
        <h4>Commentaires:</h4>
        <div v-for="comment in post.comments" :key="comment._id">
          <p>{{ comment.text }}</p>
          <button @click="deleteComment(post, comment._id!)">Supprimer Commentaire</button>
        </div>
      </div>
    </article>
    
     </div>
</template>

<style scoped>
/* --- Configuration Générale --- */
div {
  font-family: 'Inter', -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  color: #374151;
  line-height: 1.6;
  max-width: 800px;
  margin: 0 auto;
  padding: 2rem;
}

h1, h2, h3, h4 {
  color: #111827;
  font-weight: 600;
  margin-bottom: 1rem;
}

h1 { font-size: 2rem; text-align: center; margin-bottom: 2rem; }
h2 { font-size: 1.5rem; margin-top: 2rem; border-bottom: 2px solid #f3f4f6; padding-bottom: 0.5rem; }
h3 { font-size: 1.25rem; margin: 0; }

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
  box-shadow: 0 1px 3px rgba(0,0,0,0.05);
  transition: transform 0.2s, box-shadow 0.2s;
}

article:hover {
  box-shadow: 0 4px 6px rgba(0,0,0,0.05);
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

article > div {
  background-color: #f9fafb;
  border-radius: 6px;
  padding: 1rem;
  margin-top: 1rem;
}

article > div > div {
  display: flex;
  justify-content: space-between;
  align-items: center;
  border-bottom: 1px solid #e5e7eb;
  padding: 0.5rem 0;
}

article > div > div:last-child {
  border-bottom: none;
}

article > div p {
  margin: 0;
  font-size: 0.9rem;
}

div > div {
  margin-bottom: 1rem;
}
</style>