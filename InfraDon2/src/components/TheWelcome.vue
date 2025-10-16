 <script setup lang="ts">
import { ref } from 'vue';
import PouchDB from 'pouchdb'
import { onMounted } from 'vue';


// DATA - MODEL
const counter = ref(20)

// METHODS - CONTROLLER
const increment = () => {
  counter.value++
}

const storage = ref();
const postsData = ref([]);


onMounted(() => {
  console.log('=> Composant initialisé');
  initDatabase()
});

const initDatabase = () => {
  console.log('=> Connexion à la base de données');
  const db = new PouchDB('http://admin:admin@localhost:5984/infra_53_1527')
  if (db) {
    console.log("Connected to collection : " + db?.name)
    storage.value = db
  } else {
    console.warn('Something went wrong')
  }
}



</script>

<template>
  <h1>Hello World</h1>
  <button @click="increment">Increment</button>
  <p style="color: white; font-size: 50px;">{{ counter }}</p>
</template>

