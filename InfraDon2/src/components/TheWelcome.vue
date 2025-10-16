<script setup lang="ts">
import { ref } from 'vue';
import PouchDB from 'pouchdb'
import { onMounted } from 'vue';

declare interface Post {
  _id?: string;
  post_name: string;
  post_content: string;
  comments?: Comment[];
}

// DATA - MODEL
const storage = ref();
const postsData = ref<Post[]>([])

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:root@localhost:5984/dbinfradon2gab')
  if (db) {
    console.log("Connected to collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}

// Ajouter un document avec données par défaut
const addPost = () => {
  const newPost: Post = {
    post_name: 'Chillout',
    post_content: 'Je suis un gars trop chill',
    comments: []
  };

  storage.value
    .post(newPost)
    .then((response: any) => {
      console.log('Document ajouté :', response);
      
      // Recharger les données
      fetchData();
    })
    .catch((error: any) => {
      console.error('Erreur lors de l\'ajout du document :', error);
      alert('Erreur : ' + error.message);
    });
};

// Récupération des données
const fetchData = (): any => {
  storage.value
    .allDocs({ include_docs: true })
    .then((result: any) => {
      console.log('=> Données récupérées :', result.rows)
      postsData.value = result.rows.map((row: any) => row.doc);
    })
    .catch((error: any) => {
      console.error('=> Erreur lors de la récupération des données :', error)
    })
};

onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
  fetchData()
});

</script>

<template>
  <div>
    <h1>Fetch Data</h1>
    
    <!-- Bouton pour ajouter un post -->
    <button @click="addPost" style="padding: 10px 20px; background-color: #007bff; color: white; border: none; border-radius: 5px; cursor: pointer; margin-bottom: 30px;">
      Ajouter un post
    </button>

    <!-- Affichage des posts -->
    <h2>Posts existants</h2>
    <article v-for="post in postsData" :key="post._id">
      <h3>{{ post.post_name }}</h3>
      <p>{{ post.post_content }}</p>
    </article>
    <p v-if="postsData.length === 0">Aucune donnée trouvée</p>
  </div>
</template>