<template>
  <div class="flex w-screen h-screen text-gray-700">
       <div v-if="isOffline" class="w-full absolute top-0 left-0 z-10 opacity-75 text-center py-2 bg-red-300 border-b border-red-500 text-red-700">
      Sorry, it looks like you're offline.
    </div>
    <div class="flex flex-col flex-shrink-0 w-64 border-r border-gray-300 bg-gray-100">
      <!-- Sidebar -->
      <div class="h-0 overflow-auto flex-grow">
        <div class="flex items-center h-8 px-3 group mt-4">
          <div class="flex items-center flex-grow focus:outline-none">
            <a
              class="ml-2 leading-none font-medium text-sm"
              href="#"
              @click.prevent="showAllNotes"
            >All Notes</a>
          </div>
          <button @click="addNewNote">
            <svg class="h-5 w-5" xmlns="http://www.w3.org/2000/svg" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6v6m0 0v6m0-6h6m-6 0H6" />
            </svg>
          </button>
        </div>
        <div class="mt-4">
          <a
            class="flex items-center h-8 text-sm pl-8 pr-3"
            href="#"
            v-for="note in notes"
            :key="note.created"
            @click.prevent="openNote(note)"
          >
            <span class="ml-2 leading-none">{{ new Date(note.created).toLocaleString() }}</span>
          </a>
        </div>
      </div>
    </div>
    <div class="flex flex-col flex-grow" v-if="Object.keys(activeNote).length">
      <!-- Main Content - Editor -->
      <div class="flex flex-col flex-grow overflow-auto">
        <editor-content :editor="editor" />
      </div>
     <div class="h-16 bg-gray-100 border-t border-gray-300 text-right">
        <button class="flex px-4 py-3 text-white bg-blue-500 hover:bg-blue-400" @click="saveNote">Save Note</button>
      </div>
    </div>
    <div class="flex flex-col flex-grow" v-else>
      <!-- Main Content - Notes List -->
      <div class="flex flex-col flex-grow overflow-auto">
        <div v-for="note in notes" :key="note.created">
          <div class="flex px-4 pt-3 pb-4">
            <div class="prose my-2 mx-auto">
              <div>
                <span class="ml-1 text-xs text-gray-500">Created on {{ new Date(note.created).toLocaleString() }}</span>
              </div>
              <div v-html="note.content"></div>
            </div>
          </div>
          <hr class="w-full">
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { Editor, EditorContent } from '@tiptap/vue-3';
import StarterKit from '@tiptap/starter-kit';
export default {
  name: 'App',
  components: {
    EditorContent
  },
  data() {
    return {
      editor: null,
      database: null,
      notes: [],
      activeNote: {},
      isOffline: !navigator.onLine
    }
  },
  async created() {
    this.database = await this.getDatabase();
    let notes = await this.getNotes();
    this.notes = notes.reverse();
  },
  mounted() {
    this.editor = new Editor({
      content: '',
      extensions: [
        StarterKit
      ],
      editorProps: {
        attributes: {
          class: "prose my-6 mx-auto focus:outline-none"
        }
      }
    });

     window.addEventListener('offline', () => {
      this.isOffline = true;
    });
    window.addEventListener('online', () => {
      this.isOffline = false;
      // sync up a user's data with an external api
      this.syncUserData();
    });
  },
  beforeUnmount() {
    this.editor.destroy();
  },
  methods: {
    async getDatabase() {
      return new Promise((resolve, reject) => {
        let db = window.indexedDB.open("notes", 2);
        db.onerror = e => {
          reject('Error opening the database.' , e);
        };
        db.onsuccess = e => {
          console.log('db.onsuccess', e);
          resolve(e.target.result);
        };
        db.onupgradeneeded = e => {
          console.log('db.onupgradeneeded', e);
          // e.target.result.deleteObjectStore("notes");
          e.target.result.createObjectStore("notes", { keyPath: "created" });
        };
      });
    },
    async saveNote() {
      return new Promise((resolve, reject) => {
        let noteStore = this.database.transaction('notes', 'readwrite')
          .objectStore('notes');
        let noteRequest = noteStore.get(this.activeNote.created);
        noteRequest.onerror = e => {
          reject('Error saving the note in the database.' , e);
        }
        noteRequest.onsuccess = e => {
          let note = e.target.result;
          note.content = this.editor.getHTML();
          let updateRequest = noteStore.put(note);
          updateRequest.onerror = e => {
            reject('Error storing the updated note in the database.' , e);
          }
          updateRequest.onsuccess = e => {
            console.log(e)
            let noteIndex = this.notes.findIndex(n => n.created === note.created);
            this.notes[noteIndex] = note;
            resolve();
          }
        }
      });
    },
    async getNotes() {
      return new Promise((resolve, reject) => {
        console.log(reject)
        this.database.transaction('notes')
          .objectStore('notes')
          .getAll()
          .onsuccess = e => {
            console.log('getNotes()', e.target.result);
            resolve(e.target.result);
          };
      });
    },
    openNote(note) {
      this.editor.commands.setContent(note.content);
      this.activeNote = note;
    },
    showAllNotes() {
      // clear the editor
      this.editor.commands.clearContent();
      // set the activeNote as an empty object
      this.activeNote = {};
    },
    addNewNote() {
      // add a blank note to the database
      return new Promise((resolve, reject) => {
        console.log(reject)
        let transaction = this.database.transaction('notes', 'readwrite');
        transaction.oncomplete = e => {
          console.log(e)
          resolve();
        }
        let now = new Date();
        let note = {
          content: '',
          created: now.getTime()
        };
        // add that same note to the sidebar
        this.notes.unshift(note);
        // set the activeNote as the new note
        this.activeNote = note;
        transaction.objectStore('notes').add(note);
      });
    },
     syncUserData() {
      if (this.isOffline) {
        return;
      }
      // make my api request to an external server
    }
  }
}
</script>