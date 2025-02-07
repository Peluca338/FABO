<template>
  <div class="container mx-auto p-4 sm:p-6 md:max-w-3xl lg:max-w-5xl">
    <div class="flex flex-col lg:flex-row gap-6">
      <InstallPrompt />
      
      <!-- Comunidad Section -->
      <div class="w-full lg:w-2/3 flex flex-col mx-auto">
        <div class="comunidad p-4 sm:p-6 rounded-lg shadow-lg flex-grow">
          <router-link to="/nueva-publicacion"
            class="border border-cyan-400 text-white bg-cyan-500 rounded p-2 hover:bg-cyan-400 transition text-sm sm:text-base">
            Crear nueva publicación
          </router-link>

          <div v-if="cargando">
            <Loader />
          </div>

          <transition-group name="fade" tag="div">
            <div v-for="publicacion in publicaciones" :key="publicacion.id"
              class="mt-4 mb-4 sm:mb-6 p-4 sm:p-6 border border-gray-200 rounded-lg shadow-lg hover:shadow-xl transition-all duration-200 bg-white">

              <div class="flex items-center justify-between mb-4 space-x-4">
                <div class="flex items-center space-x-4 cursor-pointer" @click="abrirModalPerfil(publicacion)">
                  <img v-if="publicacion.fotoPerfil" :src="publicacion.fotoPerfil"
                    class="rounded-full w-12 h-12 object-cover border-4 border-cyan-400 shadow-md"
                    @error="publicacion.fotoPerfil = '/img/user.png'" />
                  <div>
                    <p class="text-gray-800 font-semibold text-sm sm:text-lg">
                      Por: <span class="text-cyan-600">{{ publicacion.username }}</span>
                    </p>
                    <p class="text-gray-500 text-xs sm:text-sm">{{ publicacion.email }}</p>
                  </div>
                </div>

                <i class="fa-solid fa-message text-xl text-cyan-500 cursor-pointer hover:text-cyan-400 transition"
                  @click="iniciarChatPrivado(publicacion.userId, publicacion.username)"></i>
              </div>

              <h3 class="text-lg sm:text-xl font-semibold text-gray-800">{{ publicacion.titulo }}</h3>
              <p class="text-gray-600 mb-4 text-sm">{{ publicacion.contenido }}</p>
              <p class="text-slate-500 text-xs">{{ publicacion.fechaHora }}</p>

              <div v-if="publicacion.imagen" class="mt-4 mb-4 flex justify-center">
                <img :src="publicacion.imagen" class="w-full max-w-lg rounded-lg shadow-md object-cover">
              </div>

              <div class="flex items-center gap-2 mt-4" v-if="esPropietario(publicacion.userId)">
                <button @click="editarPublicacion(publicacion.id)"
                  class="px-3 py-1 bg-amber-500 text-gray-800 rounded-md hover:bg-amber-400 transition text-xs">
                  <i class="fa-solid fa-pen-to-square text-lg"></i> Editar
                </button>
                <button @click="eliminarPublicacion(publicacion.id)"
                  class="px-3 py-1 bg-red-600 text-white rounded-md hover:bg-red-500 transition text-xs">
                  <i class="fa-solid fa-trash text-lg"></i> Eliminar
                </button>
              </div>

              <Comentarios :publicacionId="publicacion.id" class="mt-6" />
            </div>
          </transition-group>
        </div>
      </div>
    </div>

    <ModalPerfil v-if="mostrarModalPerfil" :publicacion="publicacionSeleccionada"
      @cerrar="mostrarModalPerfil = false" />
  </div>
</template>

<script>
import Swal from 'sweetalert2';
import { getFirestore, collection, getDocs, deleteDoc, doc } from "firebase/firestore";
import { getAuth } from "firebase/auth";
import { getStorage, ref, getDownloadURL } from "firebase/storage";
import Comentarios from '../Comentarios.vue';
import ModalPerfil from '../ModalPerfil.vue';
import Loader from '../Loader.vue';
import InstallPrompt from '../InstallPrompt.vue';

export default {
  components: { Comentarios, ModalPerfil, Loader, InstallPrompt },
  data() {
    return {
      publicaciones: [],
      userId: null,
      mostrarModalPerfil: false,
      publicacionSeleccionada: null,
      cargando: true,
    };
  },
  async mounted() {
    await this.obtenerPublicaciones();
    await this.obtenerUsuario();
  },
  methods: {
    async obtenerPublicaciones() {
      const db = getFirestore();
      const publicacionesSnapshot = await getDocs(collection(db, "publicaciones"));
      const storage = getStorage();

      this.publicaciones = await Promise.all(publicacionesSnapshot.docs.map(async (docSnapshot) => {
        const data = docSnapshot.data();
        let fotoPerfil = data.fotoPerfil || "/img/user.png";
        if (fotoPerfil && !fotoPerfil.startsWith("http")) {
          try {
            fotoPerfil = await getDownloadURL(ref(storage, fotoPerfil));
          } catch (error) {
            console.error("Error al obtener la foto de perfil:", error);
          }
        }
        return { id: docSnapshot.id, ...data, fotoPerfil };
      }));

      this.publicaciones.sort((a, b) => b.createdAt.toMillis() - a.createdAt.toMillis());
      this.cargando = false;
    },
    async obtenerUsuario() {
      const auth = getAuth();
      const user = auth.currentUser;
      if (user) {
        this.userId = user.uid;
      }
    },
    esPropietario(publicacionUserId) {
      return this.userId === publicacionUserId;
    },
    editarPublicacion(publicacionId) {
      this.$router.push({ name: 'EditarPublicacion', params: { id: publicacionId } });
    },
    async eliminarPublicacion(publicacionId) {
      try {
        const resultado = await Swal.fire({
          title: '¿Estás seguro?',
          text: '¡Esta acción no se puede deshacer!',
          icon: 'warning',
          showCancelButton: true,
          confirmButtonColor: '#d33',
          cancelButtonColor: '#3085d6',
          confirmButtonText: 'Sí, eliminar',
          cancelButtonText: 'Cancelar'
        });

        if (resultado.isConfirmed) {
          await deleteDoc(doc(getFirestore(), "publicaciones", publicacionId));
          this.publicaciones = this.publicaciones.filter(p => p.id !== publicacionId);
          Swal.fire('¡Eliminado!', 'La publicación ha sido eliminada.', 'success');
        }
      } catch (error) {
        console.error("Error al eliminar la publicación: ", error);
        Swal.fire('Error', 'Hubo un problema al eliminar la publicación.', 'error');
      }
    },
    iniciarChatPrivado(destinatarioId, destinatarioNombre) {
      this.$router.push({ name: 'ChatPrivado', params: { destinatarioId, destinatarioNombre } });
    },
    abrirModalPerfil(publicacion) {
      this.publicacionSeleccionada = publicacion;
      this.mostrarModalPerfil = true;
    }
  },
}
</script>

<style scoped>
.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s;
}

.fade-enter,
.fade-leave-to {
  opacity: 0;
}

.container {
  width: 100%;
  max-width: 100%;
}

.comunidad {
  background-color: #f7fafc;
  border-radius: 12px;
}

h2 {
  font-size: 2rem;
  margin-bottom: 1rem;
  color: #282828;
  font-weight: bold;
  text-shadow: 1px 1px 2px rgba(0, 0, 0, 0.3);
  letter-spacing: 0.05em;
}

h3 {
  font-size: 1.4rem;
  color: #111827;
}

img {
  max-width: 100%;
}

.text-cyan-600 {
  color: #0891B2;
}

@media (max-width: 1024px) {
  .lg\:max-w-5xl {
    max-width: 100%;
  }
}

@media (max-width: 640px) {
  .text-lg {
    font-size: 1.5rem;
  }

  .text-base {
    font-size: 0.950rem;
  }

  .text-sm {
    font-size: 0.90rem;
  }

  .p-4 {
    padding: 0.90rem;
  }

  .px-4 {
    padding-left: 1rem;
    padding-right: 1rem;
  }

  .py-2 {
    padding-top: 0.8rem;
    padding-bottom: 0.8rem;
  }
}
</style>
