<template>
  <div class="container mx-auto p-4">
    <h1 class="text-2xl font-bold mb-4">Gestion des vêtements</h1>

    <!-- Statut de synchronisation -->
    <div class="mb-4 p-4 rounded" :class="syncStatusClass">
      <p class="font-semibold">{{ syncStatusMessage }}</p>
      <p v-if="pendingChanges > 0" class="text-sm">
        {{ pendingChanges }} modification(s) en attente de synchronisation
      </p>
    </div>

    <!-- Contrôles de synchronisation -->
    <div class="mb-4 flex gap-2">
      <button @click="syncLocalToRemote" class="bg-green-500 text-white px-4 py-2 rounded">
        Synchroniser Local → Distant
      </button>
      <button @click="syncRemoteToLocal" class="bg-blue-500 text-white px-4 py-2 rounded">
        Synchroniser Distant → Local
      </button>
    </div>

    <!-- Barre de recherche et contrôles -->
    <div class="mb-4 space-y-4">
      <!-- Recherche et boutons de gestion -->
      <div class="flex gap-2">
        <input
          v-model="searchQuery"
          type="text"
          placeholder="Rechercher par nom..."
          class="border p-2 rounded flex-grow"
        />
        <button @click="generateDocs" class="bg-purple-500 text-white px-4 py-2 rounded">
          Générer 50 articles
        </button>
        <button @click="deleteAllDocs" class="bg-red-500 text-white px-4 py-2 rounded">
          Tout supprimer
        </button>
      </div>

      <!-- Formulaire d'ajout/modification -->
      <div class="border p-4 rounded">
        <h2 class="text-xl font-semibold mb-2">
          {{ isEditing ? 'Modifier le vêtement' : 'Ajouter un vêtement' }}
        </h2>
        <form @submit.prevent="submitForm" class="space-y-4">
          <div>
            <label class="block mb-1">Nom</label>
            <input
              v-model="currentDoc.nom"
              type="text"
              required
              class="border p-2 w-full rounded"
            />
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

    <!-- Historique de synchronisation -->
    <div class="mb-4">
      <h2 class="text-xl font-semibold mb-2">Historique de synchronisation</h2>
      <ul class="space-y-2">
        <li
          v-for="(log, index) in syncLogs"
          :key="index"
          class="p-2 rounded"
          :class="{
            'bg-green-100': log.type === 'local_to_remote',
            'bg-blue-100': log.type === 'remote_to_local',
            'bg-red-100': log.type === 'error'
          }"
        >
          {{ log.message }} - {{ new Date(log.timestamp).toLocaleString() }}
        </li>
      </ul>
    </div>

    <!-- Liste des vêtements -->
    <div class="mb-6">
      <h2 class="text-xl font-semibold mb-2">Liste des vêtements ({{ documents.length }})</h2>
      <div class="grid gap-4">
        <div v-for="doc in documents" :key="doc._id" class="border p-4 rounded">
          <div class="flex justify-between items-center">
            <div>
              <h3 class="font-bold">{{ doc.nom }}</h3>
              <p>Prix: {{ doc.prix }}€</p>
              <p>Taille: {{ doc.taille }}</p>
              <p class="text-sm text-gray-500">
                Dernière modification:
                {{ new Date(doc.updatedAt || doc.createdAt).toLocaleString() }}
              </p>
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
  </div>
</template>

<script>
import PouchDB from 'pouchdb'
import pouchDBFind from 'pouchdb-find'
PouchDB.plugin(pouchDBFind)

export default {
  data() {
    return {
      localDb: null,
      remoteDb: null,
      documents: [],
      currentDoc: {
        nom: '',
        prix: '',
        taille: 'M'
      },
      isEditing: false,
      editingId: null,
      syncStatus: 'idle',
      syncLogs: [],
      pendingChanges: 0,
      searchQuery: ''
    }
  },

  computed: {
    syncStatusMessage() {
      switch (this.syncStatus) {
        case 'idle':
          return 'Synchronisation inactive'
        case 'syncing':
          return 'Synchronisation en cours...'
        case 'error':
          return 'Erreur de synchronisation'
        default:
          return 'État inconnu'
      }
    },
    syncStatusClass() {
      return {
        'bg-gray-100': this.syncStatus === 'idle',
        'bg-blue-100': this.syncStatus === 'syncing',
        'bg-red-100': this.syncStatus === 'error'
      }
    }
  },

  methods: {
    async initDatabases() {
      // Base de données locale
      this.localDb = new PouchDB('vetements_local')

      // Base de données distante
      this.remoteDb = new PouchDB('http://127.0.0.1:5984/vetements', {
        fetch: function (url, opts) {
          opts.headers.set('Authorization', 'Basic ' + btoa('admin:UgEBqKGJ$P@4wS'))
          return PouchDB.fetch(url, opts)
        }
      })

      // Synchronisation continue avec méthode sync
      this.sync = PouchDB.sync(this.localDb, this.remoteDb, {
        live: true,
        retry: true
      })
        .on('change', (info) => {
          console.log('Changement sync:', info)
          this.fetchDocuments()
        })
        .on('error', (err) => {
          console.error('Erreur de synchronisation:', err)
        })

      // Écouter les changements locaux
      this.localDb
        .changes({
          since: 'now',
          live: true,
          include_docs: true
        })
        .on('change', (change) => {
          this.handleDatabaseChange(change)
        })

      // Créer un index sur le nom
      try {
        await this.localDb.put({
          _id: '_design/search',
          views: {
            by_name: {
              map: function (doc) {
                if (doc.nom) {
                  emit(doc.nom.toLowerCase())
                }
              }.toString()
            }
          }
        })
        console.log('Index créé avec succès')
      } catch (error) {
        if (error.name !== 'conflict') {
          console.error("Erreur lors de la création de l'index:", error)
        }
      }
    },

    async syncLocalToRemote() {
      try {
        this.syncStatus = 'syncing'

        const localDocs = await this.localDb.allDocs({ include_docs: true })

        for (let row of localDocs.rows) {
          const doc = row.doc

          if (doc._id && !doc._id.startsWith('_')) {
            try {
              // Utiliser replicate ou sync pour une synchronisation plus robuste
              await this.localDb.replicate.to(this.remoteDb, {
                doc_ids: [doc._id]
              })
            } catch (putError) {
              // Gérer spécifiquement les conflits de revision
              if (putError.name === 'conflict') {
                console.log('Conflit détecté, résolution nécessaire')
              } else {
                console.error('Erreur de synchronisation:', putError)
              }
            }
          }
        }

        this.addSyncLog({
          type: 'local_to_remote',
          message: `Synchronisation Local → Distant terminée (${localDocs.rows.length} documents)`
        })

        this.syncStatus = 'idle'
      } catch (error) {
        console.error('Erreur de synchronisation:', error)

        this.addSyncLog({
          type: 'error',
          message: `Erreur de synchronisation: ${error.message}`
        })

        this.syncStatus = 'error'
      }
    },

    async syncRemoteToLocal() {
      try {
        this.syncStatus = 'syncing'

        // Récupérer tous les documents distants
        const remoteDocs = await this.remoteDb.allDocs({ include_docs: true })

        // Synchroniser chaque document
        for (let row of remoteDocs.rows) {
          const doc = row.doc

          try {
            // Tenter de mettre à jour ou créer le document local
            if (doc._id && !doc._id.startsWith('_')) {
              await this.localDb.put(doc)
            }
          } catch (putError) {
            console.error("Erreur lors de la synchronisation d'un document:", putError)
          }
        }

        // Mettre à jour les documents locaux
        await this.fetchDocuments()

        // Ajouter un log de synchronisation
        this.addSyncLog({
          type: 'remote_to_local',
          message: `Synchronisation Distant → Local terminée (${remoteDocs.rows.length} documents)`
        })

        this.syncStatus = 'idle'
      } catch (error) {
        console.error('Erreur de synchronisation distante vers locale:', error)

        this.addSyncLog({
          type: 'error',
          message: `Erreur de synchronisation Distant → Local: ${error.message}`
        })

        this.syncStatus = 'error'
      }
    },

    addSyncLog(log) {
      // Ajouter un horodatage au log
      const logWithTimestamp = {
        ...log,
        timestamp: new Date().toISOString()
      }

      // Ajouter le log au début du tableau
      this.syncLogs.unshift(logWithTimestamp)

      // Limiter le nombre de logs
      if (this.syncLogs.length > 10) {
        this.syncLogs = this.syncLogs.slice(0, 10)
      }
    },

    handleDatabaseChange(change) {
      if (change.deleted) {
        this.documents = this.documents.filter((doc) => doc._id !== change.id)
      } else {
        const index = this.documents.findIndex((doc) => doc._id === change.id)
        if (index !== -1) {
          this.documents.splice(index, 1, change.doc)
        } else {
          this.documents.push(change.doc)
        }
      }
    },

    async fetchDocuments() {
      try {
        if (this.searchQuery) {
          const startkey = this.searchQuery.toLowerCase()
          const endkey = startkey + '\ufff0'

          const result = await this.localDb.query('search/by_name', {
            startkey: startkey,
            endkey: endkey,
            include_docs: true
          })

          this.documents = result.rows.map((row) => row.doc)
        } else {
          const result = await this.localDb.allDocs({ include_docs: true, descending: true })
          this.documents = result.rows.map((row) => row.doc)
        }
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
        const timestamp = new Date().toISOString()

        if (this.isEditing) {
          const doc = await this.localDb.get(this.editingId)
          await this.localDb.put({
            ...doc,
            nom: this.currentDoc.nom,
            prix: this.currentDoc.prix,
            taille: this.currentDoc.taille,
            updatedAt: timestamp
          })
        } else {
          await this.localDb.put({
            ...this.currentDoc,
            _id: new Date().toISOString(), // Générer un ID unique
            createdAt: timestamp,
            updatedAt: timestamp
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
        await this.localDb.remove(doc)
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
    },

    cleanup() {
      if (this.sync) {
        this.sync.cancel()
      }
    },

    async generateDocs() {
      try {
        const bulkDocs = []
        const types = ['T-shirt', 'Pantalon', 'Chemise', 'Veste', 'Pull', 'Short', 'Robe', 'Jupe']
        const marques = ['Nike', 'Adidas', 'Zara', 'H&M', 'Uniqlo', 'Levis']
        const couleurs = ['Rouge', 'Bleu', 'Vert', 'Noir', 'Blanc', 'Gris']
        const tailles = ['XS', 'S', 'M', 'L', 'XL']

        for (let i = 0; i < 10; i++) {
          const type = types[Math.floor(Math.random() * types.length)]
          const marque = marques[Math.floor(Math.random() * marques.length)]
          const couleur = couleurs[Math.floor(Math.random() * couleurs.length)]

          bulkDocs.push({
            _id: new Date().toISOString() + '-' + i,
            nom: `${marque} ${type} ${couleur}`,
            prix: Math.floor(Math.random() * 200) + 20,
            taille: tailles[Math.floor(Math.random() * tailles.length)],
            createdAt: new Date().toISOString(),
            updatedAt: new Date().toISOString()
          })
        }

        await this.localDb.bulkDocs(bulkDocs)
        await this.fetchDocuments()

        this.addSyncLog({
          type: 'local_to_remote',
          message: `Génération de 10 articles terminée`
        })
      } catch (error) {
        console.error('Erreur lors de la génération:', error)
        this.addSyncLog({
          type: 'error',
          message: `Erreur lors de la génération: ${error.message}`
        })
      }
    },

    async deleteAllDocs() {
      if (!confirm('Êtes-vous sûr de vouloir supprimer tous les documents ?')) {
        return
      }

      try {
        const result = await this.localDb.allDocs()
        const deleteDocs = result.rows.map((row) => ({
          _id: row.id,
          _rev: row.value.rev,
          _deleted: true
        }))

        await this.localDb.bulkDocs(deleteDocs)

        this.addSyncLog({
          type: 'local_to_remote',
          message: `Suppression de tous les documents (${deleteDocs.length} documents)`
        })

        await this.fetchDocuments()
      } catch (error) {
        console.error('Erreur lors de la suppression:', error)
        this.addSyncLog({
          type: 'error',
          message: `Erreur lors de la suppression: ${error.message}`
        })
      }
    }
  },

  mounted() {
    this.initDatabases()
  },

  beforeUnmount() {
    this.cleanup()
  },

  watch: {
    searchQuery: {
      handler() {
        this.fetchDocuments()
      },
      debounce: 300
    }
  }
}
</script>
