<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Gestion des Vêtements</h1>

    <!-- Liste des documents -->
    <div class="mb-6">
      <h2 class="text-xl font-semibold mb-2">Liste des vêtements</h2>
      <div class="grid gap-4">
        <div v-for="doc in documents" :key="doc._id" class="border p-4 rounded">
          <div class="flex justify-between items-center">
            <div>
              <h3 class="font-bold">{{ doc.nom }}</h3>
              <p>Prix: {{ doc.prix }}€</p>
              <p>Taille: {{ doc.taille }}</p>
            </div>
            <div>
              <button @click="editDoc(doc)" class="bg-blue-500 text-white px-3 py-1 rounded mr-2">
                Modifier
              </button>
              <button @click="deleteDoc(doc)" class="bg-red-500 text-white px-3 py-1 rounded">
                Supprimer
              </button>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Formulaire d'ajout/modification -->
    <div class="mb-6 border p-4 rounded">
      <h2 class="text-xl font-semibold mb-2">
        {{ isEditing ? 'Modifier le vêtement' : 'Ajouter un vêtement' }}
      </h2>
      <form @submit.prevent="submitForm" class="space-y-4">
        <div>
          <label class="block mb-1">Nom</label>
          <input v-model="currentDoc.nom" type="text" required class="border p-2 w-full rounded" />
        </div>
        <div>
          <label class="block mb-1">Prix</label>
          <input
            v-model="currentDoc.prix"
            type="number"
            required
            class="border p-2 w-full rounded"
          />
        </div>
        <div>
          <label class="block mb-1">Taille</label>
          <select v-model="currentDoc.taille" required class="border p-2 w-full rounded">
            <option value="XS">XS</option>
            <option value="S">S</option>
            <option value="M">M</option>
            <option value="L">L</option>
            <option value="XL">XL</option>
          </select>
        </div>
        <div class="flex gap-2">
          <button type="submit" class="bg-green-500 text-white px-4 py-2 rounded">
            {{ isEditing ? 'Mettre à jour' : 'Ajouter' }}
          </button>
          <button
            type="button"
            @click="generateFakeDoc"
            class="bg-purple-500 text-white px-4 py-2 rounded"
          >
            Générer démo
          </button>
        </div>
      </form>
    </div>
  </div>
</template>

<script>
import PouchDB from 'pouchdb'

export default {
  data() {
    return {
      db: null,
      documents: [],
      currentDoc: {
        nom: '',
        prix: '',
        taille: 'M'
      },
      isEditing: false,
      editingId: null
    }
  },

  methods: {
    initDatabase() {
      this.db = new PouchDB('http://127.0.0.1:5984/vetements', {
        fetch: function (url, opts) {
          opts.headers.set('Authorization', 'Basic ' + btoa('admin:UgEBqKGJ$P@4wS'))
          return PouchDB.fetch(url, opts)
        }
      })
      this.fetchDocuments()
    },

    async fetchDocuments() {
      try {
        const result = await this.db.allDocs({ include_docs: true, descending: true })
        this.documents = result.rows.map((row) => row.doc)
      } catch (error) {
        console.error('Erreur lors de la récupération des documents:', error)
      }
    },

    generateFakeDoc() {
      const tailles = ['XS', 'S', 'M', 'L', 'XL']
      const vetements = ['T-shirt', 'Pantalon', 'Chemise', 'Veste', 'Pull']

      this.currentDoc = {
        nom:
          vetements[Math.floor(Math.random() * vetements.length)] +
          ' ' +
          Math.floor(Math.random() * 100),
        prix: Math.floor(Math.random() * 100) + 10,
        taille: tailles[Math.floor(Math.random() * tailles.length)]
      }
    },

    async submitForm() {
      try {
        if (this.isEditing) {
          const doc = await this.db.get(this.editingId)
          await this.db.put({
            ...doc,
            nom: this.currentDoc.nom,
            prix: this.currentDoc.prix,
            taille: this.currentDoc.taille
          })
        } else {
          await this.db.post({
            ...this.currentDoc,
            createdAt: new Date().toISOString()
          })
        }

        await this.fetchDocuments()
        this.resetForm()
      } catch (error) {
        console.error('Erreur lors de la sauvegarde:', error)
      }
    },

    editDoc(doc) {
      this.isEditing = true
      this.editingId = doc._id
      this.currentDoc = {
        nom: doc.nom,
        prix: doc.prix,
        taille: doc.taille
      }
    },

    async deleteDoc(doc) {
      try {
        await this.db.remove(doc)
        await this.fetchDocuments()
      } catch (error) {
        console.error('Erreur lors de la suppression:', error)
      }
    },

    resetForm() {
      this.currentDoc = {
        nom: '',
        prix: '',
        taille: 'M'
      }
      this.isEditing = false
      this.editingId = null
    }
  },

  mounted() {
    this.initDatabase()
  }
}
</script>
