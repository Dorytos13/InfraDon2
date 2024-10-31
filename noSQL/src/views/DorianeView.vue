<!-- 
<script lang="ts">
import PouchDB from 'pouchdb';
export default {
  data() {
    return {
      datas: [] as any,
      databaseReference: null as PouchDB.Database | null,
    };
  },

  methods: {
   // Initialisation de la base de données
   initDatabase() {
     const db = new PouchDB('http://localhost:5984/commentaires-database'); //Aller sur l'URL : http://localhost:5174/doriane sur le navigateur inspecter avec la console
     this.databaseReference = db;
     if (db) {
      console.log("ok")
}
     // Récupération des documents après l'initialisation
     this.fetchData();
   },
   // Récupération des documents depuis la base de données
   fetchData() {
     if (!this.databaseReference) {
       console.error("Base de données non initialisée !");
       return;
     }
     // Utilisation de allDocs pour récupérer tous les documents
     this.databaseReference.allDocs({ include_docs: true }) // include_docs: true permet de récupérer le contenu complet des documents
       .then((result) => {
         // Les documents sont dans result.rows (tableau de documents)
         this.datas = result.rows.map(row => row.doc); // Stocker les docs dans datas
         console.log("Documents récupérés :", result);
       })
       .catch((error: any) => {
         console.error("Erreur lors de la récupération des documents :", error);
       });
   },
  },
  mounted() {
    this.initDatabase()
  }
}
</script>
<template>
  <h1>InfraDon2</h1>
  <p>Counter: {{ datas }}</p>
  <button type="button" role="button" @click="inc">
    +1
  </button> -->
  <!-- <div>{{ datas }}</div>
</template> -->
<script lang="ts">
import { ref } from 'vue';
import PouchDB from 'pouchdb'

declare interface Post {
  _id: string,
  doc: {
    post_name: string,
    post_content: string,
    attributes: {
      creation_date: string
    }
  }
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

    putDocument(document: Post) {
      const db = ref(this.storage).value;
      if (db) {
        db.put(document).then(() => {
          console.log('Add ok');
        }).catch((error) => {
          console.log('Add ko', error);
        })
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
          self.postsData = result.rows;
        }.bind(this)).catch(function (error: any) {
          console.log('fetchData error', error);
        });
      }
    },

    initDatabase() {
      const db = new PouchDB('http://localhost:5984/commentaires-database');
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
  <h1>Nombre de post: {{ postsData.length }}</h1>
  <ul>
    <li v-for="post in postsData" :key="post._id">
      <div class="ucfirst">{{ post.doc.post_name }}<em style="font-size: x-small;"
          v-if="post.doc.attributes?.creation_date">
          - {{ post.doc.attributes?.creation_date }}
        </em>
      </div>
    </li>
  </ul>
</template>