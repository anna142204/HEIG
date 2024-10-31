<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  //je garde la structure des premier documents faits
  _id: string
  _rev?: string
  post_name: string
  post_content: string
  attributes: {
    creation_date: string
    author: string
  }
  comments: Array<{
    comment: string
    author: string
  }>
}

export default {
  data() {
    return {
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
      selectedPost: null as Post | null
    }
  },

  mounted() {
    this.initDatabase()
    this.fetchData()
  },

  methods: {
    deleteDocument(id: string, rev: string) {
    const db = ref(this.storage).value;
    if (db) {
      db.remove(id, rev).then(() => {
        console.log('Delete ok');
        this.fetchData(); // Mettre à jour la liste des posts après la suppression
      }).catch((error) => {
        console.log('Delete ko', error);
      });
    } else {
      console.warn("Database not initialized");
    }
  },
    
    selectPostForEdit(post: Post) {
      this.selectedPost = { ...post } //copie le document pour pas modifier directement
    },

    submitEdit() {
      if (this.selectedPost) {
        this.updateDocument(this.selectedPost)  //modifie le document
        this.selectedPost = null 
      }
    },

    cancelEdit() {
      this.selectedPost = null 
    },
  
    updateDocument(document: Post) {
      const db = ref(this.storage).value
      if (db) {
        db.put(document)
          .then(() => {
            console.log('Update ok')
            this.fetchData()
          })
          .catch((error) => {
            console.log('Update ko', error)
          })
      } else {
        console.warn('Database not initialized')
      }
    },

    addSamplePost() {
      // ajouter un post fictif
      const examplePost: Post = {
        _id: new Date().toISOString(),
        post_name: 'Exemple de post',
        post_content: 'Ceci est un contenu exemple.',
        attributes: {
          creation_date: new Date().toISOString(),
          author: 'Main author'
        },
        comments: [
          { comment: 'Comment 1', author: 'Author 1' },
          { comment: 'Comment 2', author: 'Author 2' },
          { comment: 'Comment 3', author: 'Author 3' }
        ]
      }
      this.putDocument(examplePost)
    },

    putDocument(document: Post) {
      const db = ref(this.storage).value
      if (db) {
        db.put(document)
          .then(() => {
            console.log('Add ok')
            this.fetchData()
          })
          .catch((error) => {
            console.log('Add ko', error)
          })
      } else {
        console.warn('Database not initialized')
      }
    },

    fetchData() {
      const storage = ref(this.storage)
      const self = this
      if (storage.value) {
        storage.value
          .allDocs({
            include_docs: true,
            attachments: true
          })
          .then(
            function (result: any) {
              console.log('fetchData success', result)
              self.postsData = result.rows.map((row: any) => row.doc)
            }.bind(this)
          )
          .catch(function (error: any) {
            console.log('fetchData error', error)
          })
      }
    },

    initDatabase() {
      const db = new PouchDB('http://anna:admin@localhost:5984/post')
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      this.storage = db
    }
  }
}
</script>

<template>
  <ul>
    <li v-for="post in postsData" :key="post._id">
      <div class="ucfirst">
        {{ post.post_name }}
        <em style="font-size: x-small" v-if="post.attributes?.creation_date">
          - {{ post.attributes.creation_date }}
        </em>
      </div>
      <ul v-if="post.comments && post.comments.length">
        <li v-for="(comment, index) in post.comments" :key="index">
          <strong>{{ comment.author }}:</strong> {{ comment.comment }}
        </li>
      </ul>
      <button @click="selectPostForEdit(post)">Modifier le post</button>
      <button @click="deleteDocument(post._id, post._rev)">Supprimer le post</button>
    </li>
  </ul>
  <div v-if="selectedPost" class="edit-form">
    <h2>Modifier le post</h2>
    <form @submit.prevent="submitEdit">
      <label>
        Nom du post:
        <input v-model="selectedPost.post_name" />
      </label>
      <label>
        Contenu du post:
        <textarea v-model="selectedPost.post_content"></textarea>
      </label>
      <label>
        Date de création:
        <input type="text" v-model="selectedPost.attributes.creation_date" />
      </label>
      <label>
        Auteur:
        <input type="text" v-model="selectedPost.attributes.author" />
      </label>
      <button type="submit">Enregistrer les modifications</button>
      <button type="button" @click="cancelEdit">Annuler</button>
    </form>
  </div>

  <h1>Nombre de post: {{ postsData.length }}</h1>
  <button @click="addSamplePost">Ajouter un post exemple</button>
</template>
