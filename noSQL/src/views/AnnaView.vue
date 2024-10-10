<script lang="ts">
import PouchDB from 'pouchdb'
export default {
  data() {
    return {
      datas: [] as any,
      database: null as PouchDB.Database | null,
      databaseReference: null
    }
  },

  methods: {
    inc() {
      // old
      // this.total++;
    },

    initDatabase() {
      const db = new PouchDB('http://anna:admin@localhost:5984/post')
      if (db) {
        console.log("Connected to collection 'post'")
      } else {
        console.warn('Something went wrong')
      }
      this.database = db
    },

    fetchData() {
      if (!this.database) {
        console.error('Database not initialized')
        return
      }
      try {
        const result = this.database.allDocs({ include_docs: true })

        this.database
          .allDocs({
            include_docs: true
          })
          .then((result) => {
            console.log('hello', result)
            this.datas = result.rows.map((row) => row.doc) // Stocker les documents dans datas

            console.log('Données stockées dans datas :', this.datas)
            console.table(this.datas)
            // handle result
          })
          .catch(function (err) {
            console.log(err)
          })

        // this.datas = result.rows.map((row) => row.doc)
        console.log('Données récupérées :', result)
      } catch (error) {
        console.error('Erreur lors de la récupération des données:', error)
      }
    }
  },

  mounted() {
    this.initDatabase()
    this.fetchData()
  }
}
</script>

<template>
  <h1>InfraDon2</h1>

  <!-- Affichage des données récupérées -->
  <ul v-if="datas.length">
    <li v-for="(data, index) in datas" :key="index">
      <p><strong>Nom du post :</strong> {{ data.post_name }}</p>
      <p><strong>Contenu du post :</strong> {{ data.post_content }}</p>

      <!-- Affichage des attributs -->
      <p><strong>Date de création :</strong> {{ data.attributes.creation_date }}</p>
      <p><strong>Auteur :</strong> {{ data.attributes.author }}</p>

      <!-- Affichage des commentaires -->
      <ul>
        <li v-for="(comment, index) in data.comments" :key="index">
          <p><strong>Commentaire :</strong> {{ comment.comment }}</p>
          <p><strong>Auteur du commentaire :</strong> {{ comment.author }}</p>
        </li>
      </ul>
    </li>
  </ul>
  <p v-else>Aucune donnée trouvée.</p>
</template>
