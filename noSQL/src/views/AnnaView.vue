<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'
import PouchDBFind from 'pouchdb-find'

PouchDB.plugin(PouchDBFind)

declare interface Post {
  _id: string
  _rev: string
  post_name: string
  post_content: string
  attributes: {
    creation_date: string
    author: string
  }
  media: Media[]
}

declare interface Media {
  file?: File
  url: string
  type: 'image'
}

export default {
  data() {
    return {
      remoteDB: 'http://anna:admin@localhost:5984/post',
      postsData: [] as Post[],
      db: null as PouchDB.Database | null,
      editingPost: null as Post | null,
      searchTitle: '',
      selectedFile: null as File | null
    }
  },

  mounted() {
    this.initDatabase()
    this.createTitleIndex()
    this.fetchData()
  },

  methods: {
    async handleImageUpload(event: Event, postId: string) {
      const fileInput = event.target as HTMLInputElement
      if (!fileInput.files || !fileInput.files[0]) return

      const file = fileInput.files[0]
      const reader = new FileReader()

      reader.onload = async (e) => {
        if (!e.target?.result) return

        const imageUrl = e.target.result as string
        const db = this.db
        // Spécifier explicitement le type de post
        if (!db) {
          console.error('Database not valid')
          return
        }
        try {
          const post = await db.get<Post>(postId)

          // Initialiser le tableau media s'il n'existe pas
          if (!Array.isArray(post.media)) {
            post.media = []
          }

          post.media.push({
            url: imageUrl,
            type: 'image'
          })

          await db.put(post)
          this.fetchData()

          if (this.editingPost && this.editingPost._id === postId) {
            this.editingPost = post
          }

          console.log('Media ajouté avec succès')
        } catch (error) {
          console.error("Erreur lors de l'ajout du media:", error)
        }
      }

      reader.readAsDataURL(file)
    },

    async removeMedia(postId: string, mediaIndex: number) {
      const db = ref(this.db).value
      if (!db) return

      try {
        const post = await db.get<Post>(postId)
        if (!Array.isArray(post.media)) {
          post.media = []
          return
        }

        post.media.splice(mediaIndex, 1)
        await db.put(post)

        this.fetchData()
        if (this.editingPost && this.editingPost._id === postId) {
          this.editingPost = post
        }

        console.log('Media supprimé avec succès')
      } catch (error) {
        console.error('Erreur lors de la suppression du media:', error)
      }
    },

    async generateFakePosts(count: number) {
      const db = ref(this.db).value
      if (!db) {
        console.error('Database not valid')
        return
      }

      const fakePosts: Omit<Post, '_rev'>[] = Array.from({ length: count }, (_, index) => ({
        _id: new Date().toISOString() + '-' + index,
        post_name: `Post ${index + 1}`,
        post_content: `This is the content for post ${index + 1}.`,
        attributes: {
          creation_date: new Date().toISOString(),
          author: `Author ${index + 1}`
        },
        media: [
          {
            url: 'https://picsum.photos/300/201',
            type: 'image' as const
          }
        ]
      }))

      try {
        await db.bulkDocs(fakePosts)
        console.log(`Successfully added ${count} fake posts`)
        this.fetchData()
      } catch (error) {
        console.error('Error generating fake posts', error)
      }
    },

    fetchData() {
  const db = ref(this.db).value;
  if (!db) {
    console.error('Database not valid');
    return;
  }

  db.allDocs({
    include_docs: true,
    attachments: true, 
  })
    .then((result: any) => {
      this.postsData = result.rows
        .filter((row: any) => !row.id.startsWith('_design/'))
        .map((row: any) => row.doc);
    })
    .catch((error) => {
      console.error('Error fetching data:', error);
    });
}
,
    putDocument() {
      const document = this.generateFakePosts(1)
      const db = ref(this.db).value
      if (db) {
        db.put
          .bind(this)(document)
          .then(() => {
            console.log('Add ok')
            this.fetchData()
          })
          .catch((error) => {
            console.log('Add ko', error)
          })
      }
    },

    editPost(post: Post) {
      // Crée une copie pour éviter de modifier directement `postsData`
      this.editingPost = { ...post }
    },

    cancelEdit() {
      // Annule l'édition en réinitialisant `editingPost`
      this.editingPost = null
    },

    async savePost() {
      const db = ref(this.db).value
      if (!db || !this.editingPost) return

      try {
        const post = await db.get(this.editingPost._id)
        this.editingPost._rev = post._rev

        await db.put(this.editingPost)
        console.log('Post modifié avec succès')
        this.fetchData()
        this.editingPost = null
      } catch (error) {
        console.error('Erreur lors de la modification du post', error)
      }
    },

    async deleteData(id: string, rev: string) {
      console.log('Call deleteData', rev)
      const db = ref(this.db).value
      if (!db) {
        console.log('Database not valid')
        return
      }
      try {
        await db.remove(id, rev)
        console.log('deleteData success')
        this.fetchData()
      } catch (error) {
        console.log('deleteData error', error)
      }
    },

    initDatabase() {
      const localDB = 'post'
      const $db = new PouchDB(localDB)
      if ($db) {
        console.log('Connected to collection: ' + $db.name)
      } else {
        console.warn('Something went wrong')
      }
      this.db = $db
    },

    updateLocalDatabase() {
      const db = ref(this.db).value
      if (db) {
        db.replicate.from
          .bind(this)(this.remoteDB)
          .on('complete', () => {
            console.log('on replicate complete')
            this.fetchData()
          })
          .on('error', function (error) {
            console.log('error', error)
          })
      }
    },

    updateDistantDatabase() {
      console.log('updateDistantDatabase called')
      const db = ref(this.db).value
      if (db) {
        db.replicate
          .to(this.remoteDB)
          .on('complete', () => console.log('Replication completed'))
          .on('error', (err: any) => console.error('Replication error', err))
      }
    },

    watchRemoteDatabase() {
      console.log('watchRemoteDatabase called')
      const db = ref(this.db).value
      if (db) {
        db.sync(this.remoteDB, { live: true, retry: true })
          .on('change', (info: any) => console.log('Change detected', info))
          .on('error', (err: any) => console.error('Sync error', err))
      }
    },

    async createTitleIndex() {
      const db = ref(this.db).value
      if (!db) {
        console.error('Database not valid')
        return
      }
      try {
        // Liste des index existants
        const existingIndexes = await db.getIndexes()
        const indexExists = existingIndexes.indexes.some(
          (index) => index.name === 'post_name_index'
        )

        if (!indexExists) {
          // Crée l'index uniquement s'il n'existe pas
          await db.createIndex({
            index: {
              fields: ['post_name'],
              name: 'post_name_index' // Nom explicite de l'index
            }
          })
          console.log('Index on "post_name" created successfully')
        } else {
          console.log('Index on "post_name" already exists')
        }
      } catch (error) {
        console.error('Error ensuring index:', error)
      }
    },

    async searchByTitle(title: string) {
      const db = ref(this.db).value
      if (!db) {
        console.error('Database not valid')
        return
      }

      try {
        const result = await db.find({
          selector: {
            post_name: { $regex: new RegExp(`.*${title}.*`, 'i') } // Recherche exacte
          }
        })
        console.log('Search results:', result.docs)
        this.postsData = result.docs as Post[] // Met à jour les données affichées
      } catch (error) {
        console.error('Error searching by title:', error)
      }
    },

    resetSearch() {
      this.searchTitle = '' // Réinitialiser le champ de recherche
      this.fetchData() // Recharger tous les documents
    }
  }
}
</script>

<template>
  <div>
    <!-- Liste des posts -->
    <h1>Liste des posts</h1>
    <!-- Recherche filtrée -->
    <div class="search-container">
      <label for="title-search">Rechercher par titre :</label>
      <input id="title-search" v-model="searchTitle" placeholder="Entrez un titre" />
      <button @click="searchByTitle(searchTitle)" class="search-btn">Rechercher</button>
      <button @click="resetSearch" class="reset-btn">Réinitialiser</button>
    </div>
    <li v-for="post in postsData" :key="post._id" class="post-item">
      <h2 class="post-title">{{ post.post_name }}</h2>

      <!-- Conteneur pour médias et contenu -->
      <div class="post-layout">
        <!-- Section média -->
        <div v-if="post.media && post.media.length" class="post-media">
          <div v-for="(mediaItem, index) in post.media" :key="index" class="media-item">
            <img v-if="mediaItem.type === 'image'" :src="mediaItem.url" alt="Post Image" />
            <button @click="removeMedia(post._id, index)" class="remove-media-btn">
              Enlever le média
            </button>
          </div>
        </div>

        <!-- Contenu du post -->
        <div class="post-content">
          <p>{{ post.post_content }}</p>
        </div>
      </div>

      <!-- Actions -->
      <button @click="deleteData(post._id, post._rev)" class="delete-btn">Supprimer</button>
      <button @click="editPost(post)" class="edit-btn">Modifier</button>
      <!-- Formulaire d'édition, affiché uniquement pour le post sélectionné -->
      <div v-if="editingPost && editingPost._id === post._id" class="edit-form">
        <h2>Modifier le post</h2>
        <form @submit.prevent="savePost">
          <label>
            Nom du post:
            <input v-model="editingPost.post_name" placeholder="Nom du post" />
          </label>

          <label>
            Contenu du post:
            <textarea v-model="editingPost.post_content" placeholder="Contenu"></textarea>
          </label>
          <label>
            Ajouter un média:
            <input type="file" @change="handleImageUpload($event, editingPost?._id)" />
          </label>
          <div class="edit-buttons">
            <button type="submit" class="save-btn">Enregistrer</button>
            <button type="button" @click="cancelEdit" class="cancel-btn">Annuler</button>
          </div>
        </form>
      </div>
    </li>

    <button @click="putDocument" class="add-post-btn">Ajouter un post exemple</button>
    <button @click="generateFakePosts(10)" class="add-post-btn">Générer 10 posts exemples</button>
    <div>
      <button @click="updateLocalDatabase" class="sync-btn">Mettre à jour la base locale</button>
      <button @click="updateDistantDatabase" class="sync-btn">
        Mettre à jour la base distante
      </button>
      <button @click="watchRemoteDatabase" class="watch-btn">Surveiller la base distante</button>
    </div>

    <h1>Nombre de posts: {{ postsData.length }}</h1>
  </div>
</template>
<style>
body {
  font-family: Arial, sans-serif;
  margin: 0;
  padding: 0;
  background-color: #f4f4f4;
  color: #333;
}

.container {
  max-width: 900px;
  margin: 20px auto;
  padding: 20px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

h1,
h2 {
  text-align: center;
  color: #007bff;
}

/* Liste des posts */
.post-item {
  border: 1px solid #ddd;
  padding: 20px;
  margin: 20px 0;
  border-radius: 10px;
  background-color: #fff;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.1);
  transition:
    transform 0.2s,
    box-shadow 0.2s;
  list-style: none;
}

.post-item:hover {
  transform: translateY(-5px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.2);
}

/* Titre du post */
.post-title {
  font-weight: bold;
  font-size: 1.5em;
  margin-bottom: 10px;
  color: #007bff;
}

.creation-date {
  font-size: 0.9em;
  color: #666;
}
/* Conteneur principal des médias et du contenu */
.post-layout {
  display: flex;
  gap: 20px; /* Espacement entre l'image et le contenu */
  align-items: flex-start; /* Aligner les éléments en haut */
}

/* Section média */
.post-media {
  flex-shrink: 0; /* Empêche la section média de rétrécir */
  max-width: 200px; /* Limite la largeur des médias */
}

.post-media img {
  max-width: 100%; /* Assure que les images ne dépassent pas la largeur de leur conteneur */
  border-radius: 5px;
  object-fit: cover;
}

/* Section contenu */
.post-content {
  flex: 1; /* Prend tout l'espace disponible */
  font-size: 1em;
  line-height: 1.6;
  color: #444;
}

/* Actions */
.post-actions {
  margin-top: 15px;
  display: flex;
  gap: 10px;
}
.remove-media-btn {
  background-color: #acacac;
  color: white;
  border: none;
  padding: 8px 12px;
  border-radius: 4px;
  cursor: pointer;
  display: flex;
}

button {
  padding: 8px 12px;
  margin: 5px;
  border-radius: 4px;
  border: none;
  cursor: pointer;
}

button:hover {
  opacity: 0.9;
}

.add-post-btn {
  background-color: #007bff;
  color: white;
}

.delete-btn {
  background-color: #dc3545;
  color: white;
}

.edit-btn {
  background-color: #ccc;
  color: white;
}

.sync-btn {
  background-color: #17a2b8;
  color: white;
}

.watch-btn {
  background-color: #ffc107;
  color: black;
}

.edit-form {
  margin-top: 20px;
  padding: 20px;
  border: 1px solid #ccc;
  border-radius: 6px;
  background-color: #fff;
}

.edit-form label {
  display: block;
  font-weight: bold;
  margin-bottom: 8px;
}

.edit-form input,
.edit-form textarea {
  width: 100%;
  padding: 10px;
  margin-bottom: 15px;
  border: 1px solid #ddd;
  border-radius: 4px;
}

.edit-buttons {
  display: flex;
  gap: 10px;
}

.save-btn {
  background-color: #28a745;
  color: white;
}

.cancel-btn {
  background-color: #dc3545;
  color: white;
}
/* Conteneur principal pour la recherche */
.search-container {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
  align-items: center;
  justify-content: center;
  margin-bottom: 20px;
  padding: 10px;
  background-color: #f8f9fa; /* Fond gris clair */
  border-radius: 8px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.1);
}

/* Style du label */
.search-container label {
  font-size: 16px;
  font-weight: bold;
  margin-right: 10px;
  color: #333;
}

/* Style de l'input */
.search-container input {
  padding: 10px;
  font-size: 16px;
  border: 1px solid #ddd;
  border-radius: 5px;
  flex: 1; /* S'étend dans l'espace disponible */
  max-width: 300px; /* Limite la largeur */
}

/* Style du bouton de recherche */
.search-container .search-btn {
  background-color: #007bff; /* Bleu vif */
  color: white;
  border: none;
  padding: 10px 16px;
  border-radius: 5px;
  font-size: 16px;
  cursor: pointer;
  transition:
    background-color 0.3s ease,
    box-shadow 0.3s ease;
}

.search-container .search-btn:hover {
  background-color: #0056b3; /* Bleu foncé */
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}

/* Style du bouton de réinitialisation */
.search-container .reset-btn {
  background-color: #6c757d; /* Gris */
  color: white;
  border: none;
  padding: 10px 16px;
  border-radius: 5px;
  font-size: 16px;
  cursor: pointer;
  transition:
    background-color 0.3s ease,
    box-shadow 0.3s ease;
}

.search-container .reset-btn:hover {
  background-color: #5a6268; /* Gris foncé */
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.2);
}
</style>
