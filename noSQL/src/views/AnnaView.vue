<script lang="ts">
import { ref } from 'vue'
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string
  _rev: string
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
      remoteDB: 'http://anna:admin@localhost:5984/post',
      postsData: [] as Post[],
      db: null as PouchDB.Database | null
    }
  },

  mounted() {
    this.initDatabase()
    this.fetchData()
  },

  methods: {
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

    getFakePost() {
      return {
        _id: new Date().toISOString(),
        post_name: 'post_name' + new Date().toISOString(),
        post_content: 'post_content',
        attributes: {
          creation_date: new Date().toISOString()
        }
      }
    },

    putDocument() {
      const document = this.getFakePost()
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

    fetchData() {
      const db = ref(this.db).value
      if (db) {
        db.allDocs
          .bind(this)({
            include_docs: true,
            attachments: true
          })
          .then((result: any) => {
            console.log('fetchData success =>', result.rows)
            this.postsData = result.rows.map((row: any) => row.doc)
          })
          .catch(function (error: any) {
            console.log('fetchData error', error)
          })
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
      
     },

    watchRemoteDatabase() {
      
    }
  }
}
</script>

<template>
  <div>
    <!-- Liste des posts -->
    <h1>Liste des posts</h1>
    <ul>
      <li v-for="post in postsData" :key="post._id" class="post-item">
        <div class="post-title">
          <strong>{{ post.post_name }}</strong>
          <em class="creation-date" v-if="post.attributes?.creation_date">
            - {{ post.attributes.creation_date }}
          </em>
        </div>

        <p>{{ post.post_content }}</p>

        <!-- Liste des commentaires -->
        <ul class="comments" v-if="post.comments && post.comments.length">
          <li v-for="(comment, index) in post.comments" :key="index">
            <strong>{{ comment.author }}:</strong> {{ comment.comment }}
          </li>
        </ul>

        <!-- Actions -->
        <button @click="deleteData(post._id, post._rev)" class="delete-btn">Supprimer</button>
      </li>
    </ul>

    <!-- Section pour ajouter un post fictif -->
    <button @click="putDocument" class="add-post-btn">Ajouter un post exemple</button>
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
