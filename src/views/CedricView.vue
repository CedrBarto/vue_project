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
    addToBtn: function () {
      this.btnValue += 1
    },
    initDatabase: function () {
      try {
        let bdd = new PouchDB('http://admin:UgEBqKGJ$P@4wS@127.0.0.1:5984/vetements')
        console.log('Base de données initialisée :', bdd)
        this.db = bdd
      } catch (error) {
        console.error("Erreur lors de l'initialisation de la base de données :", error)
      }
    },
    getAllDocs: function () {
      this.db.allDocs({ include_docs: true, descending: true }, function (err, doc) {
        console.log(doc.rows)
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
