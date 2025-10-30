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

      fetchData();
    })

};

const deletePost = (post: Post) => {
  storage.value
    .remove(post)
    .then(() => {
      fetchData();
    })
};

const updatePost = (post: Post) => {
  storage.value
    .get(post._id)
    .then((doc: any) => {
      doc.post_name = "Modifié !";  
      doc.post_content = new Date().toISOString();
      return storage.value.put(doc);
    })
    .then(() => {
      fetchData();
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
    <button @click="addPost">
      Ajouter un post
    </button>

    <!-- Affichage des posts -->
    <h2>Posts existants</h2>
    <article v-for="post in postsData" :key="post._id">
      <h3>{{ post.post_name }}</h3>
      <p>{{ post.post_content }}</p>
      <button @click="deletePost(post)">
        Supprimer ce post
      </button>
      <button @click="updatePost(post)">
        Modifier ce post
      </button>
    </article>
    <p v-if="postsData.length === 0">Aucune donnée trouvée</p>
  </div>
</template>