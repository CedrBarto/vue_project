<template>
  <div class="ex">
    <h1>This is my page</h1>
    <button @click="getAllDocs">{{ btnValue }}</button>
  </div>
</template>

<script>
import PouchDB from 'Pouchdb'

export default {
  data() {
    return {
      btnValue: 0,
      db: null
    }
  },
  methods: {
    addToBtn() {
      this.btnValue += 1
    },
    initDatabase() {
      // Créez une instance de PouchDB sans inclure les informations d'identification dans l'URL
      const db = new PouchDB('http://127.0.0.1:5984/vetements', {
        fetch: function (url, opts) {
          // Ajoutez un en-tête d'authentification avec les informations d'identification
          opts.headers.set('Authorization', 'Basic ' + btoa('admin:UgEBqKGJ$P@4wS'))
          return PouchDB.fetch(url, opts)
        }
      })

      console.log('Base de données initialisée :', db)
      this.db = db
    },
    getAllDocs() {
      this.db.allDocs({ include_docs: true, descending: true }, function (doc, err) {
        console.log(doc, err)
      })
    }
  },
  mounted() {
    this.initDatabase()
  }
}
</script>

<style>
@media (min-width: 1024px) {
  .ex {
    min-height: 100vh;
    display: flex;
    align-items: center;
  }
}
</style>
