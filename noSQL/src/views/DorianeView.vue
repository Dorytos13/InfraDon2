<script lang="ts">
import { ref } from 'vue';
import PouchDB from 'pouchdb';
import PouchDBFind from 'pouchdb-find'
PouchDB.plugin(PouchDBFind)

declare interface Media {
  url: string;
}

declare interface Comment {
  comment: string;
  author: string;
}

declare interface Post {
  _id: string;
  _rev?: string;
  post_name: string;
  post_content: string;
  attributes: {
    creation_date: string;
    author: string;
  },
  comments: Comment[];
  media: Media[];
}

export default {
  data() {
    return {
      remoteDB: 'http://Dory:Admin13@localhost:5984/commentaires-database',
      postsData: [] as Post[],
      storage: null as PouchDB.Database | null,
      changes: null as any,
      isEditing: false,
      isAdding: false,
      previewPostId: null as string | null,
      searchTitle: '',
      addForm: {
        post_name: '',
        post_content: '',
        attributes: {
          author: ''
        },
        comments: [] as Comment[],
        media: [] as Media[]
      },
      editForm: {
        post_name: '',
        post_content: '',
        _id: '',
        _rev: '',
        attributes: {
          author: ''
        },
        comments: [] as Comment[],
        media: [] as Media[]
      }
    };
  },

  mounted() {
    this.initDatabase();
    this.createTitleIndex();
    this.fetchData();
    this.watchRemoteDatabase();
  },

  beforeUnmount() {
    if (this.changes) {
      this.changes.cancel();
    }
  },

  methods: {
    // Initialisation de la base de données
    initDatabase() {
      const localDB = 'local-commentaires';
      const $db = new PouchDB(localDB);
      if ($db) {
        console.log('Connected to collection: ' + $db.name);
      } else {
        console.warn('Something went wrong');
      }
      this.storage = $db;
    },

    // Récupération des données
    fetchData() {
      const db = ref(this.storage).value;
      if (db) {
        db.allDocs({
          include_docs: true,
          attachments: true
        }).then((result: any) => {
          console.log('fetchData success =>', result.rows);
          this.postsData = result.rows.map((row: any) => ({
            _id: row.id,
            _rev: row.doc._rev,
            post_name: row.doc.post_name,
            post_content: row.doc.post_content,
            attributes: {
              creation_date: row.doc.attributes?.creation_date || '',
              author: row.doc.attributes?.author || ''
            },
            comments: row.doc.comments || []
          }));
        }).catch((error: any) => {
          console.log('fetchData error', error);
        });
      }
    },

    async createTitleIndex() {
      const db = ref(this.storage).value
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
      const db = ref(this.storage).value
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
    },

    generateFakePosts() {
      const fakePosts: Post[] = [];
      for (let i = 0; i < 10; i++) {
        const fakePost: Post = {
          _id: Date.now().toString() + i, // Utilisation de l'heure actuelle pour garantir un ID unique
          post_name: `Post ${i + 1}: ${this.randomString(10)}`, // Génération d'un titre aléatoire
          post_content: `Voici un contenu aléatoire pour le post numéro ${i + 1}. Il peut contenir n'importe quel texte.`,
          attributes: {
            creation_date: new Date().toISOString(),
            author: `Auteur ${this.randomString(5)}`
          },
          comments: [], // Aucun commentaire pour le moment
          media: [] // Aucun média pour le moment
        };
        fakePosts.push(fakePost);
      }

      const db = ref(this.storage).value;
      if (db) {
        const batch = fakePosts.map(post => db.put(post)); // Mise en lot des ajouts
        Promise.all(batch).then(() => {
          console.log('10 faux posts ajoutés');
          this.fetchData();
        }).catch((error) => {
          console.log('Erreur lors de l\'ajout des faux posts', error);
        });
      }
    },

    // Fonction pour générer une chaîne de caractères aléatoire
    randomString(length: number): string {
      const characters = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789';
      let result = '';
      for (let i = 0; i < length; i++) {
        result += characters.charAt(Math.floor(Math.random() * characters.length));
      }
      return result;
    },

    // Méthodes de synchronisation
    updateLocalDatabase() {
      const db = ref(this.storage).value;
      if (db) {
        db.replicate.from(this.remoteDB)
          .on('complete', () => {
            console.log('on replicate complete');
            this.fetchData();
          })
          .on('error', function (error) {
            console.log('error', error);
          });
      }
    },

    updateDistantDatabase() {
      const db = ref(this.storage).value;
      if (db) {
        db.replicate.to(this.remoteDB)
          .on('complete', () => {
            console.log('Synchronisation avec la base distante terminée');
          })
          .on('error', (error) => {
            console.log('Erreur lors de la synchronisation:', error);
          });
      }
    },

    watchRemoteDatabase() {
      const db = ref(this.storage).value;
      if (db) {
        this.changes = db.changes({
          since: 'now',
          live: true,
          include_docs: true
        })
        .on('change', (change) => {
          console.log('Changement détecté:', change);
          this.fetchData();
        })
        .on('error', (error) => {
          console.log('Erreur de watch:', error);
        });
      }
    },

    //méthode pour la synchronisation bidirectionnelle
    async syncBothDatabases() {
      const db = ref(this.storage).value;
      if (db) {
        try {
          // Synchronisation depuis le serveur vers local
          await db.replicate.from(this.remoteDB)
            .on('complete', () => {
              console.log('Synchronisation depuis le serveur terminée');
            })
            .on('error', (error) => {
              console.log('Erreur lors de la synchronisation depuis le serveur:', error);
            });

          // Synchronisation depuis local vers le serveur
          await db.replicate.to(this.remoteDB)
            .on('complete', () => {
              console.log('Synchronisation vers le serveur terminée');
            })
            .on('error', (error) => {
              console.log('Erreur lors de la synchronisation vers le serveur:', error);
            });

          // Actualiser les données après la synchronisation complète
          this.fetchData();
          console.log('Synchronisation bidirectionnelle terminée');
        } catch (error) {
          console.log('Erreur lors de la synchronisation bidirectionnelle:', error);
        }
      }
    },

    // Méthodes CRUD
    startAdd() {
      this.isAdding = true;
      this.isEditing = false;
    },

    addDocument() {
      const newDoc: Post = {
        _id: Date.now().toString(), // Utilisation d'un timestamp comme ID temporaire
        post_name: this.addForm.post_name,
        post_content: this.addForm.post_content,
        attributes: {
          creation_date: new Date().toISOString(),
          author: this.addForm.attributes.author
        },
        comments: this.addForm.comments,
        media: this.addForm.media
      };

      const db = ref(this.storage).value;
      if (db) {
        db.put(newDoc).then(() => {
          console.log('Add ok');
          this.fetchData();
          // this.updateDistantDatabase(); // Synchroniser après l'ajout
          this.cancelAdd();
        }).catch((error) => {
          console.log('Add ko', error);
        });
      }
    },

    cancelAdd() {
      this.isAdding = false;
      this.addForm = {
        post_name: '',
        post_content: '',
        attributes: { author: '' },
        comments: [],
        media: []
      };
    },

    addComment() {
      this.addForm.comments.push({ comment: '', author: '' });
    },

    deleteComment(index: number) {
      this.addForm.comments.splice(index, 1);
    },

    // Nouvelle méthode pour ajouter un commentaire dans le formulaire d'édition
    addEditComment() {
      this.editForm.comments.push({ comment: '', author: '' });
    },

    // Nouvelle méthode pour supprimer un commentaire dans le formulaire d'édition
    deleteEditComment(index: number) {
      this.editForm.comments.splice(index, 1);
    },

    // Modification de la méthode startEdit pour s'assurer que comments est initialisé
    startEdit(post: Post) {
      this.editForm = {
        post_name: post.post_name,
        post_content: post.post_content,
        _id: post._id,
        _rev: post._rev || '',
        attributes: {
          author: post.attributes.author
        },
        comments: [...(post.comments || [])], // S'assure que comments est cloné correctement
        media: [...(post.media || [])]
      };
      this.isEditing = true;
      this.isAdding = false;
    },

    deleteDocument(post: Post) {
      const db = ref(this.storage).value;
      if (db && post._id && post._rev) {
        console.log('Deleting document:', post);
        db.remove(post._id, post._rev).then(() => {
          console.log('Delete ok');
          this.fetchData();
          // this.updateDistantDatabase(); // Synchroniser après la suppression
        }).catch((error) => {
          console.log('Delete ko', error);
        });
      }
    },

    saveEdit() {
      const db = ref(this.storage).value;
      if (db && this.editForm._id && this.editForm._rev) {
        const updatedDoc: Post = {
          _id: this.editForm._id,
          _rev: this.editForm._rev,
          post_name: this.editForm.post_name,
          post_content: this.editForm.post_content,
          attributes: {
            creation_date: new Date().toISOString(),
            author: this.editForm.attributes.author
          },
          comments: this.editForm.comments,
          media: this.editForm.media
        };

        db.put(updatedDoc).then(() => {
          console.log('Update ok');
          this.fetchData();
          this.cancelEdit();
          // this.updateDistantDatabase(); // Synchroniser après la mise à jour
          this.cancelEdit();
        }).catch((error) => {
          console.log('Update ko', error);
        });
      }
    },

    cancelEdit() {
      this.isEditing = false;
      this.editForm = {
        post_name: '',
        post_content: '',
        _id: '',
        _rev: '',
        attributes: { author: '' },
        comments: [],
        media: []
      };
    },

    previewPost(postId: string) {
      this.previewPostId = this.previewPostId === postId ? null : postId;
    },

    addMedia() {
      this.addForm.media.push({ url: ''});
    },

    deleteMedia(index: number) {
      this.addForm.media.splice(index, 1);
    },

    addEditMedia() {
      this.editForm.media.push({ url: ''});
    },

    deleteEditMedia(index: number) {
      this.editForm.media.splice(index, 1);
    },

  }
};
</script>

<template>
  <div class="app-container">
    <header class="header">
      <h1 class="title">Gestion des Posts ({{ postsData.length }})</h1>
      <div class="search-container">
        <label for="title-search">Rechercher par titre :</label>
        <input id="title-search" v-model="searchTitle" placeholder="Entrez un titre de post" />
        <button @click="searchByTitle(searchTitle)" class="search-btn">Rechercher</button>
        <button @click="resetSearch" class="reset-btn">Réinitialiser</button>
      </div>
      
      <div class="sync-buttons">
        <button class="btn sync" @click="updateLocalDatabase">
          <span class="icon">↓</span> Synchroniser depuis le serveur
        </button>
        <button class="btn sync" @click="updateDistantDatabase">
          <span class="icon">↑</span> Synchroniser vers le serveur
        </button>
        <button class="btn sync-both" @click="syncBothDatabases">
          <span class="icon">↕</span> Synchronisation bidirectionnelle
        </button>
      </div>
    </header>

    <main class="main-content">
      <button class="btn add-btn" @click="startAdd">+ Ajouter un document</button>
      <button class="btn generate-fake-posts-btn" @click="generateFakePosts">Générer 10 faux posts</button>

      <ul class="posts-list">
        <li v-for="post in postsData" :key="post._id" class="post-card">
          <div class="post-header">
            <h2 class="post-title ucfirst">{{ post.post_name }}</h2>
            <div class="post-meta">
              <em v-if="post.attributes.creation_date">
                Créé le: {{ new Date(post.attributes.creation_date).toLocaleDateString() }}
              </em>
            </div>
          </div>
          
          <div class="media-section">
            <h3>Médias</h3>
            <div v-for="(media, index) in post.media" :key="index" class="media-item">
              <img :src="media.url" :alt="'Media ' + (index + 1)" />
            </div>
          </div>

          <div class="post-content">{{ post.post_content }}</div>

          <div v-if="previewPostId === post._id" class="post-preview">
          <div v-if="post.comments.length > 0">
            <h3>Commentaires</h3>
            <ul>
              <li v-for="(comment, index) in post.comments" :key="index">
                <p><strong>{{ comment.author }}:</strong> {{ comment.comment }}</p>
              </li>
            </ul>
          </div>
          <div v-else>
            <p>Aucun commentaire pour ce post.</p>
          </div>
        </div>
          
          <div class="post-actions">
            <button class="btn preview" @click="previewPost(post._id)">
              {{ previewPostId === post._id ? 'Fermer l\'aperçu' : 'Aperçu' }}
            </button>
            <button class="btn edit" @click="startEdit(post)">Modifier</button>
            <button class="btn delete" @click="deleteDocument(post)">Supprimer</button>
          </div>
        </li>
      </ul>
    </main>

    <!-- Formulaire d'ajout -->
    <div v-if="isAdding" class="modal">
      <div class="modal-content">
        <h2>Ajouter un nouveau document</h2>
        <form @submit.prevent="addDocument" class="form">
          <div class="form-group">
            <label for="add_post_name">Nom du post :</label>
            <input type="text" v-model="addForm.post_name" id="add_post_name" required />
          </div>

          <div class="form-group">
            <label for="add_post_content">Contenu du post :</label>
            <textarea v-model="addForm.post_content" id="add_post_content" required></textarea>
          </div>

          <div class="form-group">
            <label for="add_author">Auteur :</label>
            <input type="text" v-model="addForm.attributes.author" id="add_author" required />
          </div>

          <div class="media-section">
            <h3>Médias</h3>
            <div v-for="(media, index) in addForm.media" :key="index" class="media-form">
              <div class="form-group">
                <label :for="'media_url' + index">URL :</label>
                <input type="text" v-model="media.url" :id="'media_url' + index" required />
              </div>


              <button type="button" class="btn delete small" @click="deleteMedia(index)">
                Supprimer le média
              </button>
            </div>
            <button type="button" class="btn add-media" @click="addMedia">
              + Ajouter un média
            </button>
          </div>

          <div class="comments-section">
            <h3>Commentaires</h3>
            <div v-for="(comment, index) in addForm.comments" :key="index" class="comment-form">
              <div class="form-group">
                <label :for="'comment' + index">Commentaire :</label>
                <input type="text" v-model="comment.comment" :id="'comment' + index" required />
              </div>

              <div class="form-group">
                <label :for="'author' + index">Auteur du commentaire :</label>
                <input type="text" v-model="comment.author" :id="'author' + index" required />
              </div>

              <button type="button" class="btn delete small" @click="deleteComment(index)">
                Supprimer le commentaire
              </button>
            </div>
            <button type="button" class="btn add-comment" @click="addComment">
              + Ajouter un commentaire
            </button>
          </div>

          <div class="form-actions">
            <button type="submit" class="btn submit">Enregistrer</button>
            <button type="button" class="btn cancel" @click="cancelAdd">Annuler</button>
          </div>
        </form>
      </div>
    </div>
  </div>
    <!-- Formulaire de modification -->
  <div v-if="isEditing" class="modal">
    <div class="modal-content">
      <h2>Modifier le document</h2>
      <form @submit.prevent="saveEdit" class="form">
        <div class="form-group">
          <label for="edit_post_name">Nom du post :</label>
          <input type="text" v-model="editForm.post_name" id="edit_post_name" required />
        </div>

        <div class="form-group">
          <label for="edit_post_content">Contenu du post :</label>
          <textarea v-model="editForm.post_content" id="edit_post_content" required></textarea>
        </div>

        <div class="form-group">
          <label for="edit_author">Auteur :</label>
          <input type="text" v-model="editForm.attributes.author" id="edit_author" required />
        </div>

        <div class="media-section">
          <h3>Médias</h3>
          <div v-for="(media, index) in editForm.media" :key="index" class="media-form">
            <div class="form-group">
              <label :for="'edit_media_url' + index">URL :</label>
              <input type="text" v-model="media.url" :id="'edit_media_url' + index" required />
            </div>

            <button type="button" class="btn delete small" @click="deleteEditMedia(index)">
              Supprimer le média
            </button>
          </div>
          <button type="button" class="btn add-media" @click="addEditMedia">
            + Ajouter un média
          </button>
        </div>

        <div class="comments-section">
          <h3>Commentaires</h3>
          <div v-for="(comment, index) in editForm.comments" :key="index" class="comment-form">
            <div class="form-group">
              <label :for="'edit_comment' + index">Commentaire :</label>
              <input type="text" v-model="comment.comment" :id="'edit_comment' + index" required />
            </div>

            <div class="form-group">
              <label :for="'edit_author_comment' + index">Auteur du commentaire :</label>
              <input type="text" v-model="comment.author" :id="'edit_author_comment' + index" required />
            </div>

            <button type="button" class="btn delete small" @click="deleteEditComment(index)">
              Supprimer le commentaire
            </button>
          </div>
          <button type="button" class="btn add-comment" @click="addEditComment">
            + Ajouter un commentaire
          </button>
        </div>

        <div class="form-actions">
          <button type="submit" class="btn submit">Enregistrer</button>
          <button type="button" class="btn cancel" @click="cancelEdit">Annuler</button>
        </div>
      </form>
    </div>
  </div>
</template>

<style scoped>
/* Variables CSS */
:root {
  --primary-color: #3498db;
  --danger-color: #e74c3c;
  --success-color: #2ecc71;
  --warning-color: #f1c40f;
  --text-color: #2c3e50;
  --background-color: #ecf0f1;
  --card-background: #ffffff;
  --border-radius: 8px;
  --spacing: 1rem;
}

/* Styles de base */
.app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
  font-family: Arial, sans-serif;
  color: var(--text-color);
}

.header {
  margin-bottom: 2rem;
  text-align: center;
}

.title {
  color: var(--primary-color);
  margin-bottom: 1rem;
}
.post-preview {
  background: var(--background-color);
  padding: var(--spacing);
  border-radius: var(--border-radius);
  margin-top: var(--spacing);
  font-style: italic;
  color: var(--text-color);
  font-size: 1em;
}

.search-container {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 1rem;
  max-width: 600px;
  margin: 0 auto;
}

.search-container label {
  font-size: 1rem;
  color: white;
  white-space: nowrap;
}

#title-search {
  flex: 1;
  padding: 0.5rem 1rem;
  border: 2px solid #e0e0e0;
  border-radius: 4px;
  font-size: 1rem;
  transition: border-color 0.3s ease;
}

#title-search:focus {
  outline: none;
  border-color: #4a90e2;
  box-shadow: 0 0 5px rgba(74, 144, 226, 0.3);
}

#title-search::placeholder {
  color: #999;
}

.search-btn {
  padding: 0.5rem 1.2rem;
  background-color: brown;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.3s ease;
}


.search-btn:active {
  transform: translateY(1px);
}

.reset-btn {
  padding: 0.5rem 1.2rem;
  background-color: #95a5a6;
  color: white;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

/* Boutons */
.btn {
  padding: 8px 16px;
  border: none;
  border-radius: var(--border-radius);
  cursor: pointer;
  font-weight: bold;
  transition: all 0.3s ease;
  margin: 0 5px;
}

.btn:hover {
  opacity: 0.9;
  transform: translateY(-1px);
}

.sync {
  background-color: var(--primary-color);
  color: white;
  border: 2px solid brown;
}

.edit {
  background-color: var(--warning-color);
  color: var(--text-color);
}

.delete {
  background-color: var(--danger-color);
  color: white;
}

.submit {
  background-color: var(--success-color);
  color: white;
}

.cancel {
  background-color: #95a5a6;
  color: white;
}

.add-btn {
  background-color: var(--success-color);
  color: white;
  font-size: 1.1em;
  margin: 1rem 0;
}

/* Liste des posts */
.posts-list {
  list-style: none;
  padding: 0;
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 20px;
}

.post-card {
  background: var(--card-background);
  border-radius: var(--border-radius);
  padding: var(--spacing);
  box-shadow: 0 2px 5px rgba(0,0,0,0.1);
  transition: transform 0.3s ease;
}

.post-card:hover {
  transform: translateY(-3px);
}

.post-title {
  margin: 0;
  color: var(--primary-color);
  font-size: 1.2em;
}

.post-meta {
  font-size: 0.9em;
  color: #666;
  margin: 5px 0;
}

.post-content {
  margin: 10px 0;
  line-height: 1.5;
}

.post-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}

/* Modal et formulaires */
.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
}

.modal-content {
  background: var(--card-background);
  padding: 2rem;
  border-radius: var(--border-radius);
  width: 90%;
  max-width: 600px;
  max-height: 90vh;
  overflow-y: auto;
}

.form-group {
  margin-bottom: 1rem;
}

.form-group label {
  display: block;
  margin-bottom: 0.5rem;
  font-weight: bold;
}

.form-group input,
.form-group textarea {
  width: 100%;
  padding: 8px;
  border: 1px solid #ddd;
  border-radius: var(--border-radius);
  font-size: 1em;
}

.form-group textarea {
  min-height: 100px;
  resize: vertical;
}

.form-actions {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
  margin-top: 1rem;
}

/* Section commentaires */
.comments-section {
  margin: 1rem 0;
  padding: 1rem;
  background: var(--background-color);
  border-radius: var(--border-radius);
}

.comment-form {
  margin-bottom: 1rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #ddd;
}

.add-comment {
  background-color: var(--primary-color);
  color: white;
}

.sync-both {
  background-color: brown;
  color: white;
  border-radius: 10px;
}

.sync-buttons {
  display: flex;
  gap: 10px;
  justify-content: center;
  flex-wrap: wrap;
}

.btn.small {
  padding: 4px 8px;
  font-size: 0.8em;
  margin-top: 5px;
}
/* Utilitaires */
.ucfirst {
  text-transform: capitalize;
}

/* Media queries pour la responsivité */
@media (max-width: 768px) {
  .posts-list {
    grid-template-columns: 1fr;
  }

  .modal-content {
    width: 95%;
    padding: 1rem;
  }

  .btn {
    padding: 6px 12px;
    font-size: 0.9em;
  }
}
</style>