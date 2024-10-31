<script lang="ts">
import { ref } from 'vue'; // Importation de la fonction ref de Vue pour créer des références réactives
import PouchDB from 'pouchdb'; // Importation de PouchDB pour la gestion de la base de données

// Déclaration de l'interface pour un commentaire
declare interface Comment {
  comment: string; // Contenu du commentaire
  author: string; // Auteur du commentaire
}

// Déclaration de l'interface pour un post
declare interface Post {
  _id: string; // Identifiant unique 
  _rev?: string; // Révision du post (utilisé pour le versioning dans PouchDB)
  post_name: string; // Nom du post
  post_content: string; // Contenu du post
  attributes: { // Attributs supplémentaires du post
    creation_date: string; // Date de création du post
    author: string; // Auteur du post
  },
  comments: Comment[]; // Tableau de commentaires associés au post
}

export default {
  // Déclaration des données de composant
  data() {
    return {
      postsData: [] as Post[], // Tableau pour stocker les données des posts
      storage: null as PouchDB.Database | null, // Référence à la base de données PouchDB
      remoteDB: null as PouchDB.Database | null, // Référence à la base de données distante
      isEditing: false, // État pour déterminer si le formulaire est en mode édition
      isAdding: false, // État pour déterminer si le formulaire est en mode ajout
      addForm: { // Formulaire pour ajouter un nouveau post
        post_name: '', // Nom du post
        post_content: '', // Contenu du post
        attributes: { // Attributs supplémentaires pour le post
          author: '' // Auteur du post
        },
        comments: [] as Comment[] // Tableau pour stocker les commentaires
      },
      editForm: { // Formulaire pour modifier un post existant
        post_name: '', // Nom du post
        post_content: '', // Contenu du post
        _id: '', // Identifiant du post
        _rev: '', // Révision du post
        attributes: { // Attributs supplémentaires pour le post
          author: '' // Auteur du post
        },
        comments: [] as Comment[] // Tableau pour stocker les commentaires
      }
    };
  },

  // Méthode exécutée lorsque le composant est monté
  mounted() {
    this.initDatabase(); // Initialisation de la base de données
    this.fetchData(); // Récupération des données des posts existants
    this.startSync(); // Démarrer la synchronisation
  },

  methods: {
    // Méthode pour initialiser la base de données PouchDB
    initDatabase() {
      const db = new PouchDB('local-commentaires'); // Base de données locale
      const remoteDb = new PouchDB('http://localhost:5984/commentaires-database'); // Base de données distante
      this.storage = db;
      this.remoteDB = remoteDb;
    },

    // Méthode pour récupérer les données des posts dans la base de données
    fetchData() {
      const storage = ref(this.storage); // Créer une référence réactive à la base de données
      if (storage.value) { // Vérifier si la base de données est initialisée
        storage.value.allDocs({
          include_docs: true, // Inclure les documents dans le résultat
          attachments: true // Inclure les pièces jointes
        }).then((result: any) => {
          // Mapper les résultats pour les stocker dans postsData
          this.postsData = result.rows.map((row: any) => ({
            _id: row.id, // Identifier le post
            _rev: row.doc._rev, // Révision du post
            post_name: row.doc.post_name, // Nom du post
            post_content: row.doc.post_content, // Contenu du post
            attributes: {
              creation_date: row.doc.attributes?.creation_date || '', // Date de création
              author: row.doc.attributes?.author || '' // Auteur
            },
            comments: row.doc.comments || [] // Commentaires
          }));
        }).catch((error: any) => {
          console.log('fetchData error', error); // Gérer les erreurs
        });
      }
    },

    startSync() {
      if (this.storage && this.remoteDB) {
        // Synchroniser la base de données locale et distante
        const sync = PouchDB.sync(this.storage, this.remoteDB, {
          live: true, // Synchronisation en temps réel
          retry: true // Réessayer en cas de déconnexion
        }).on('change', (info) => {
          console.log('Changement détecté', info);
          this.fetchData(); // Mettre à jour les données locales à chaque changement
        }).on('error', (err) => {
          console.error('Erreur de synchronisation', err);
        });
      }
    },

    // Méthode pour démarrer l'ajout d'un nouveau post
    startAdd() {
      this.isAdding = true; // Afficher le formulaire d'ajout
      this.isEditing = false; // Assurez-vous que le formulaire d'édition est masqué
    },
 
    // Méthode pour ajouter un nouveau document
    addDocument() {
      const newDoc: Post = {
        _id: new Date().toISOString(), // Générer un nouvel ID basé sur la date
        post_name: this.addForm.post_name, // Utiliser le nom du post du formulaire
        post_content: this.addForm.post_content, // Utiliser le contenu du post du formulaire
        attributes: {
          creation_date: new Date().toISOString(), // Définir la date de création
          author: this.addForm.attributes.author // Utiliser l'auteur du formulaire
        },
        comments: this.addForm.comments // Utiliser les commentaires du formulaire
      };
      this.putDocument(newDoc); // Appeler la méthode pour ajouter le document à la base de données
      this.cancelAdd(); // Réinitialiser le formulaire d'ajout
    },
 
    // Méthode pour ajouter un document dans PouchDB
    putDocument(document: Post) {
      const db = ref(this.storage).value; // Récupérer la référence de la base de données
      if (db) { // Vérifier si la base de données est disponible
        db.put(document).then(() => {
          console.log('Add ok'); // Log pour succès
          this.fetchData(); // Rafraîchir les données après l'ajout
        // La synchronisation gère automatiquement les mises à jour de la base de données distante
        }).catch((error) => {
          console.log('Add ko', error); // Gérer les erreurs
        });
      }
    },
 
    // Méthode pour annuler l'ajout d'un post
    cancelAdd() {
      this.isAdding = false; // Masquer le formulaire d'ajout
      this.addForm = { // Réinitialiser les champs du formulaire
        post_name: '',
        post_content: '',
        attributes: { author: '' },
        comments: []
      };
    },
 
    // Méthode pour ajouter un commentaire au formulaire
    addComment() {
      this.addForm.comments.push({ comment: '', author: '' }); // Ajouter un commentaire vide au tableau
    },
 
    // Méthode pour supprimer un commentaire spécifique du formulaire
    deleteComment(index: number) {
      this.addForm.comments.splice(index, 1); // Supprimer le commentaire à l'index spécifié
    },
 
    // Méthode pour démarrer l'édition d'un post
    startEdit(post: Post) {
      this.editForm = {
        post_name: post.post_name, // Remplir le formulaire d'édition avec les données du post
        post_content: post.post_content,
        _id: post._id,
        _rev: post._rev || '',
        attributes: {
          author: post.attributes.author // Remplir l'auteur
        },
        comments: post.comments || [] // Remplir les commentaires
      };
      this.isEditing = true; // Afficher le formulaire d'édition
      this.isAdding = false; // Masquer le formulaire d'ajout
    },
 
    // Méthode pour supprimer un post
    deleteDocument(post: Post) {
      const db = ref(this.storage).value; // Récupérer la référence de la base de données
      if (db && post._id && post._rev) { // Vérifier que le post a un ID et une révision
        db.remove(post._id, post._rev).then(() => {
          console.log('Delete ok'); // Log pour succès
          this.fetchData(); // Rafraîchir les données après la suppression
        }).catch((error) => {
          console.log('Delete ko', error); // Gérer les erreurs
        });
      } else {
        console.log('Erreur : document _id ou _rev manquant pour la suppression'); // Gérer l'erreur si ID ou révision manquant
      }
    },
 
    // Méthode pour enregistrer les modifications d'un post
    saveEdit() {
      const db = ref(this.storage).value; // Récupérer la référence de la base de données
      if (db && this.editForm._id && this.editForm._rev) { // Vérifier que le formulaire d'édition a un ID et une révision
        const updatedDoc: Post = {
          _id: this.editForm._id, // Utiliser l'ID du formulaire d'édition
          _rev: this.editForm._rev, // Utiliser la révision du formulaire d'édition
          post_name: this.editForm.post_name, // Utiliser le nom du post du formulaire
          post_content: this.editForm.post_content, // Utiliser le contenu du post du formulaire
          attributes: {
            creation_date: new Date().toISOString(), // Mettre à jour la date de création
            author: this.editForm.attributes.author // Utiliser l'auteur du formulaire
          },
          comments: this.editForm.comments // Utiliser les commentaires du formulaire
        };
        
        db.put(updatedDoc).then(() => {
          this.fetchData(); // Rafraîchir les données après la mise à jour
          this.cancelEdit(); // Réinitialiser le formulaire d'édition
        }).catch((error) => {
          console.log('Erreur de mise à jour', error); // Gérer les erreurs
        });
      }
    },
 
    // Méthode pour annuler l'édition d'un post
    cancelEdit() {
      this.isEditing = false; // Masquer le formulaire d'édition
      this.editForm = { // Réinitialiser les champs du formulaire d'édition
        post_name: '',
        post_content: '',
        _id: '',
        _rev: '',
        attributes: { author: '' },
        comments: []
      };
    }
  }
};
</script>
 
<template>
  <h1>Nombre de posts : {{ postsData.length }}</h1>
  <button @click="startAdd">Ajouter un document</button>
 
  <ul>
    <li v-for="post in postsData" :key="post._id">
      <div class="ucfirst">
        {{ post.post_name }}
        <em v-if="post.attributes.creation_date">- {{ post.attributes.creation_date }}</em>
      </div>
      <button @click="startEdit(post)">Modifier</button>
      <button @click="deleteDocument(post)">Supprimer</button>
    </li>
  </ul>
 
  <!-- Formulaire d'ajout de document -->
  <div v-if="isAdding">
    <h2>Ajouter un nouveau document</h2>
    <form @submit.prevent="addDocument">
      <label for="add_post_name">Nom du post :</label>
      <input type="text" v-model="addForm.post_name" id="add_post_name" required />
 
      <label for="add_post_content">Contenu du post :</label>
      <textarea v-model="addForm.post_content" id="add_post_content" required></textarea>
 
      <label for="add_author">Auteur :</label>
      <input type="text" v-model="addForm.attributes.author" id="add_author" required />
 
      <h3>Commentaires</h3>
      <div v-for="(comment, index) in addForm.comments" :key="index">
        <label :for="'comment' + index">Commentaire :</label>
        <input type="text" v-model="comment.comment" :id="'comment' + index" required />
 
        <label :for="'author' + index">Auteur du commentaire :</label>
        <input type="text" v-model="comment.author" :id="'author' + index" required />
 
        <button type="button" @click="deleteComment(index)">Supprimer le commentaire</button>
      </div>
      <button type="button" @click="addComment">Ajouter un commentaire</button>
 
      <button type="submit">Enregistrer</button>
      <button type="button" @click="cancelAdd">Annuler</button>
    </form>
  </div>
 
  <!-- Formulaire de modification de document -->
  <div v-if="isEditing">
    <h2>Modifier le document</h2>
    <form @submit.prevent="saveEdit">
      <label for="edit_post_name">Nom du post :</label>
      <input type="text" v-model="editForm.post_name" id="edit_post_name" required />
 
      <label for="edit_post_content">Contenu du post :</label>
      <textarea v-model="editForm.post_content" id="edit_post_content" required></textarea>
 
      <label for="edit_author">Auteur :</label>
      <input type="text" v-model="editForm.attributes.author" id="edit_author" required />
 
      <h3>Commentaires</h3>
      <div v-for="(comment, index) in editForm.comments" :key="index">
        <label :for="'edit_comment' + index">Commentaire :</label>
        <input type="text" v-model="comment.comment" :id="'edit_comment' + index" required />
 
        <label :for="'edit_author_comment' + index">Auteur du commentaire :</label>
        <input type="text" v-model="comment.author" :id="'edit_author_comment' + index" required />
      </div>
 
      <button type="submit">Enregistrer</button>
      <button type="button" @click="cancelEdit">Annuler</button>
    </form>
  </div>
</template>
 