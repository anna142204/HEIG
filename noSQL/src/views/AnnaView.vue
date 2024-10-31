<script lang="ts">
import { ref } from 'vue';
import PouchDB from 'pouchdb';

declare interface Post {
  _id: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: string;
  };
}

export default {
  data() {
    return {
      total: 0,
      postsData: [] as Post[],
      document: null as Post | null,
      storage: null as PouchDB.Database | null,
    };
  },

  mounted() {
    this.initDatabase();
    this.fetchData();
  },

  methods: {

    addSamplePost() {
      const examplePost: Post = {
        _id: new Date().toISOString(),
        post_name: 'Exemple de post',
        post_content: 'Ceci est un contenu exemple.',
        attributes: {
          creation_date: new Date().toISOString(),
        }
      };
      this.putDocument(examplePost);
    },

    putDocument(document: Post) {
      const db = ref(this.storage).value;
      if (db) {
        db.put(document).then(() => {
          console.log('Add ok');
          this.fetchData();
        }).catch((error) => {
          console.log('Add ko', error);
        });
      } else {
        console.warn("Database not initialized");
      }
    },

    fetchData() {
      const storage = ref(this.storage);
      const self = this;
      if (storage.value) {
        (storage.value).allDocs({
          include_docs: true,
          attachments: true
        }).then(function (result: any) {
          console.log('fetchData success', result);
          self.postsData = result.rows.map((row: any) => row.doc);
        }.bind(this)).catch(function (error: any) {
          console.log('fetchData error', error);
        });
      }
    },

    initDatabase() {
      const db = new PouchDB('http://anna:admin@localhost:5984/post');
      if (db) {
        console.log("Connected to collection 'post'");
      } else {
        console.warn("Something went wrong");
      }
      this.storage = db;
    }
  },
}
</script>
<template>
  <ul>
    <li v-for="post in postsData" :key="post._id">
      <div class="ucfirst">
        {{ post.post_name }}
        <em style="font-size: x-small;" v-if="post.attributes?.creation_date">
          - {{ post.attributes.creation_date }}
        </em>
      </div>
    </li>
  </ul>
  <h1>Nombre de post: {{ postsData.length }}</h1>
  <button @click="addSamplePost">Ajouter un post exemple</button>
</template>