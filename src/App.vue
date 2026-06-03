<template>
  <div class="container">

    <header class="header-fijo">
      <div class="header-contenido">
        <h1>🍔 Vue Food</h1>
        <div class="acciones-header">
          <button class="btn-toggle-form" @click="mostrarFormulario = !mostrarFormulario">
            {{ mostrarFormulario ? '✕ Cerrar' : '➕ Nuevo Producto' }}
          </button>
          <div class="carrito-icono" @click="mostrarCarrito = true">
            🛒 <span class="badge" v-if="carrito.length > 0">{{ carrito.length }}</span>
          </div>
        </div>
      </div>

      <div class="filtros-bar">
        <input 
          v-model="busqueda" 
          type="text" 
          placeholder="🔍 Buscar plato..." 
          class="input-busqueda"
        />
        <select v-model="categoriaSeleccionada" class="select-categoria">
          <option value="">Todas las categorías</option>
          <option v-for="cat in categoriasUnicas" :key="cat" :value="cat">{{ cat }}</option>
        </select>
      </div>
    </header>

    <div class="header-espaciador"></div>

    <Transition name="fade">
      <div v-if="mostrarFormulario" class="formulario-container">
        <div class="formulario-card">
          <h3>➕ Agregar Nuevo Producto</h3>
          <div class="form-grid">
            <input v-model="nuevo.nombre" placeholder="Nombre del plato" />
            <input v-model="nuevo.categoria" placeholder="Categoría (Ej: Pizzas)" />
            <input v-model.number="nuevo.precio" type="number" placeholder="Precio" />
            <input v-model.number="nuevo.stock" type="number" placeholder="Stock inicial" />
            <input v-model="nuevo.descripcion" placeholder="Descripción" class="full-width" />
            <input v-model="nuevo.imagen" placeholder="URL de la imagen" class="full-width" />
          </div>
          <button class="btn-crear" @click="agregarProductoNuevo">Guardar Producto</button>
        </div>
      </div>
    </Transition>

    <div class="productos-grid">
      <div class="card" v-for="producto in productosFiltrados" :key="producto.id">
        <div class="card-img">
          <img :src="producto.imagen" :alt="producto.nombre" />
          <span class="tag-categoria">{{ producto.categoria }}</span>
        </div>
        <div class="card-info">
          <h3>{{ producto.nombre }}</h3>
          <p class="desc">{{ producto.descripcion }}</p>
          <h4 class="precio">{{ formatearMoneda(producto.precio) }}</h4>
          <p :class="['stock', { 'sin-stock': producto.stock <= 0 }]">
            Stock: {{ producto.stock }}
          </p>
          <div class="botones-card">
            <button @click="agregarCarrito(producto)" :disabled="producto.stock <= 0">
              {{ producto.stock <= 0 ? 'Agotado' : '🛒 Agregar' }}
            </button>
            <button class="btn-stock" @click="sumarStock(producto)">+ Stock</button>
          </div>
        </div>
      </div>
    </div>

    <Transition name="slide">
      <div v-if="mostrarCarrito" class="panel-carrito">
        <div class="carrito-header">
          <h2>Tu Pedido</h2>
          <button class="cerrar-panel" @click="mostrarCarrito = false">✕</button>
        </div>

        <div v-if="carrito.length === 0" class="carrito-vacio">
          <p>El carrito está vacío 🍔</p>
        </div>

        <div class="carrito-items" v-else>
          <div v-for="item in carrito" :key="item.id" class="item-carrito">
            <div class="item-detalles">
              <strong>{{ item.nombre }}</strong>
              <small>{{ item.cantidad }} x {{ formatearMoneda(item.precio) }}</small>
            </div>
            <div class="item-total">
              {{ formatearMoneda(item.precio * item.cantidad) }}
              <button @click="eliminarDelCarrito(item)">❌</button>
            </div>
          </div>
        </div>

        <div class="carrito-footer">
          <div class="fila-total">
            <span>Total:</span>
            <strong>{{ formatearMoneda(totalCarrito) }}</strong>
          </div>
          <button class="btn-finalizar" @click="finalizarCompra" :disabled="carrito.length === 0">
            Finalizar Compra
          </button>
        </div>
      </div>
    </Transition>

    <div v-if="loading" class="spinner-overlay">
      <div class="spinner"></div>
      <p>Generando factura...</p>
    </div>

    <div v-if="notificacion" class="notificacion-toast">
      {{ notificacion }}
    </div>

  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import jsPDF from "jspdf"

// ESTADOS
const mostrarCarrito = ref(false)
const mostrarFormulario = ref(false)
const loading = ref(false)
const notificacion = ref('')
const busqueda = ref('')
const categoriaSeleccionada = ref('')

// FORMULARIO NUEVO
const nuevo = ref({
  nombre: '', descripcion: '', precio: null, stock: null, categoria: '', imagen: ''
})

// PRODUCTOS (LOS 30 COMPLETOS)
const productos = ref([
  { id: 1, nombre: 'Hamburguesa Clásica', categoria: 'Hamburguesas', descripcion: 'Carne y queso', precio: 18000, stock: 10, imagen: 'https://img.hogar.mapfre.es/wp-content/uploads/2018/09/hamburguesa-sencilla.jpg' },
  { id: 2, nombre: 'Hamburguesa Doble', categoria: 'Hamburguesas', descripcion: 'Doble carne', precio: 22000, stock: 10, imagen: 'https://progcarne.com/storage/app/uploads/public/608/6d1/8b0/6086d18b065a7811052900.jpg' },
  { id: 3, nombre: 'Hamburguesa BBQ', categoria: 'Hamburguesas', descripcion: 'Salsa BBQ', precio: 20000, stock: 10, imagen: 'https://www.recetasnestlecam.com/sites/default/files/srh_recipes/74e1a2dfe688f08eedf86a3711c8e4fb.png' },
  { id: 4, nombre: 'Pizza Personal', categoria: 'Pizzas', descripcion: 'Pizza pequeña', precio: 20000, stock: 10, imagen: 'https://www.tupperware.com/cdn/shop/articles/pypavrtlteochfsubmyt-412455.jpg?v=1647285652' },
  { id: 5, nombre: 'Pizza Hawaiana', categoria: 'Pizzas', descripcion: 'Jamón y piña', precio: 22000, stock: 10, imagen: 'https://irecetasfaciles.com/wp-content/uploads/2020/03/pizza-hawaiana.jpg' },
  { id: 6, nombre: 'Pizza Pepperoni', categoria: 'Pizzas', descripcion: 'Pepperoni', precio: 23000, stock: 10, imagen: 'https://ik.imagekit.io/smithfield/armour/4353bced-f940-00d0-8c6e-13a0a4a7f5c2/2ac60829-5178-4a6e-80cf-6ca43d862cee/Quick-and-Easy-Pepperoni-Pizza-700x700.jpeg' },
  { id: 7, nombre: 'Papas Pequeñas', categoria: 'Papas', descripcion: 'Papas fritas', precio: 8000, stock: 10, imagen: 'https://houseofbpty.com/wp-content/uploads/2022/11/Baby-Potatoes-800x530.jpg' },
  { id: 8, nombre: 'Papas Grandes', categoria: 'Papas', descripcion: 'Papas fritas grandes', precio: 12000, stock: 10, imagen: 'https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhS280bw-krYPp2dukCuyXD2UFTjXCboDVPvpF33OPQcss6hQdXEk7OESRFk35p2sPrSZvKKTJSJN11nRxJp3U-aMsji9SNMFKMDHxcvKFoUx2MQ6xLXpbiF1wcGGmXd_TST84wxeDdZM1K/s1600/PATATAS+COCIDAS+Y+LUEGO+FRITAS+(3).jpg' },
  { id: 9, nombre: 'Salchipapa', categoria: 'Papas', descripcion: 'Salchicha y papas', precio: 15000, stock: 10, imagen: 'https://imag.bonviveur.com/emplatado-final-de-las-salchipapas.jpg' },
  { id: 10, nombre: 'Hot Dog', categoria: 'Hot Dogs', descripcion: 'Perro caliente', precio: 10000, stock: 10, imagen: 'https://egcexswrzkz.exactdn.com/wp-content/uploads/2026/04/perrito-caliente.jpg' },
  { id: 11, nombre: 'Hot Dog Especial', categoria: 'Hot Dogs', descripcion: 'Perro especial', precio: 14000, stock: 10, imagen: 'https://www.lavanguardia.com/files/og_thumbnail/uploads/2020/04/20/5ea0a4775e8dd.jpeg' },
  { id: 12, nombre: 'Arepa con Carne', categoria: 'Arepas', descripcion: 'Arepa rellena', precio: 12000, stock: 10, imagen: 'https://familiakitchen.com/wp-content/uploads/2021/06/Carne-mechada-apepa.jpg' },
  { id: 13, nombre: 'Arepa con Pollo', categoria: 'Arepas', descripcion: 'Arepa pollo', precio: 12000, stock: 10, imagen: 'https://comedera.com/wp-content/uploads/sites/9/2025/10/Arepa-de-pollo-colombia.webp' },
  { id: 14, nombre: 'Sandwich Jamón', categoria: 'Sandwiches', descripcion: 'Sandwich J&Q', precio: 9000, stock: 10, imagen: 'https://comedera.com/wp-content/uploads/sites/9/2021/03/sandwich-de-jamon-y-queso.jpg' },
  { id: 15, nombre: 'Sandwich Pollo', categoria: 'Sandwiches', descripcion: 'Sandwich pollo', precio: 11000, stock: 10, imagen: 'https://imag.bonviveur.com/sandwich-de-pollo.jpg' },
  { id: 16, nombre: 'Gaseosa', categoria: 'Bebidas', descripcion: 'Bebida fría', precio: 5000, stock: 10, imagen: 'https://www.aguaparatuvida.com.ar/img/novedades/106.jpg' },
  { id: 17, nombre: 'Jugo Natural', categoria: 'Bebidas', descripcion: 'Jugo natural', precio: 6000, stock: 10, imagen: 'https://k-listo.com/wp-content/uploads/2021/11/JUGOS.jpg' },
  { id: 18, nombre: 'Malteada', categoria: 'Bebidas', descripcion: 'Malteada', precio: 9000, stock: 10, imagen: 'https://i.blogs.es/c6f09d/como-hacer-malteada-chocolate-cremosa-receta-facil-mundo/840_560.jpg' },
  { id: 19, nombre: 'Café', categoria: 'Bebidas', descripcion: 'Café caliente', precio: 4000, stock: 10, imagen: 'https://upload.wikimedia.org/wikipedia/commons/4/45/A_small_cup_of_coffee.JPG' },
  { id: 20, nombre: 'Capuccino', categoria: 'Bebidas', descripcion: 'Capuccino', precio: 7000, stock: 10, imagen: 'https://upload.wikimedia.org/wikipedia/commons/thumb/0/00/Cappuccino_PeB.jpg/1280px-Cappuccino_PeB.jpg' },
  { id: 21, nombre: 'Empanada', categoria: 'Entradas', descripcion: 'Empanada colombiana', precio: 3000, stock: 10, imagen: 'https://imag.bonviveur.com/empanadas-colombianas.jpg' },
  { id: 22, nombre: 'Pastel de Pollo', categoria: 'Entradas', descripcion: 'Pastel hojaldre', precio: 5000, stock: 10, imagen: 'https://exhibirequipos.com/wp-content/uploads/2020/03/pastel-de-pollo-1.jpg' },
  { id: 23, nombre: 'Chorizo', categoria: 'Entradas', descripcion: 'Chorizo con arepa', precio: 6000, stock: 10, imagen: 'https://assets.tmecosys.com/image/upload/t_web_rdp_recipe_584x480_1_5x/img/recipe/ras/Assets/b66f872a179dea7b8d6f717b99077bf9/Derivates/5649f5b2588a2556b3e4517d13ed7a17404c5e6e.jpg' },
  { id: 24, nombre: 'Costillas BBQ', categoria: 'Especiales', descripcion: 'Costillas de cerdo', precio: 28000, stock: 10, imagen: 'https://i.blogs.es/5fc93e/1/650_1200.jpg' },
  { id: 25, nombre: 'Pollo Broaster', categoria: 'Especiales', descripcion: 'Pollo frito', precio: 25000, stock: 10, imagen: 'https://imag.bonviveur.com/pollo-broaster.jpg' },
  { id: 26, nombre: 'Alitas BBQ', categoria: 'Especiales', descripcion: 'Alitas de pollo', precio: 18000, stock: 10, imagen: 'https://www.unileverfoodsolutions.com.co/dam/global-ufs/mcos/NOLA/calcmenu/recipes/col-recipies/recetas-fruco-bbq/Alitas-BBQ-1200x709.jpg' },
  { id: 27, nombre: 'Alitas Picantes', categoria: 'Especiales', descripcion: 'Alitas picantes', precio: 18000, stock: 10, imagen: 'https://sandersonfarms.com/wp-content/uploads/2020/12/Sanderson-Farms_Kevin-Rathbuns-Spicy-Chicken-Wings_28218.jpg' },
  { id: 28, nombre: 'Helado', categoria: 'Postres', descripcion: 'Helado cremoso', precio: 6000, stock: 10, imagen: 'https://images.cookforyourlife.org/wp-content/uploads/2020/06/Chocolate-Whipped-Ice-Cream-shutterstock_1010248351.jpg' },
  { id: 29, nombre: 'Postre Tres Leches', categoria: 'Postres', descripcion: 'Postre de la casa', precio: 8000, stock: 10, imagen: 'https://cdn0.recetasgratis.net/es/posts/0/3/4/postre_de_tres_leches_frio_34430_600.jpg' },
  { id: 30, nombre: 'Brownie', categoria: 'Postres', descripcion: 'Brownie chocolate', precio: 7000, stock: 10, imagen: 'https://www.cocinista.es/download/bancorecursos/recetas/receta-brownies.jpg' }
])

const carrito = ref([])

// COMPUTADOS
const categoriasUnicas = computed(() => [...new Set(productos.value.map(p => p.categoria))])

const productosFiltrados = computed(() => {
  return productos.value.filter(p => {
    const matchBusqueda = p.nombre.toLowerCase().includes(busqueda.value.toLowerCase())
    const matchCategoria = categoriaSeleccionada.value === '' || p.categoria === categoriaSeleccionada.value
    return matchBusqueda && matchCategoria
  })
})

const totalCarrito = computed(() => carrito.value.reduce((acc, item) => acc + (item.precio * item.cantidad), 0))

// MÉTODOS
const formatearMoneda = (v) => new Intl.NumberFormat('es-CO', { style: 'currency', currency: 'COP', minimumFractionDigits: 0 }).format(v)

const lanzarNotificacion = (m) => {
  notificacion.value = m
  setTimeout(() => notificacion.value = '', 2000)
}

const agregarProductoNuevo = () => {
  if (!nuevo.value.nombre || !nuevo.value.precio) {
    lanzarNotificacion('⚠️ Nombre y Precio requeridos')
    return
  }
  const id = Date.now()
  productos.value.push({ id, ...nuevo.value })
  lanzarNotificacion('✅ Producto Agregado')
  nuevo.value = { nombre: '', descripcion: '', precio: null, stock: null, categoria: '', imagen: '' }
  mostrarFormulario.value = false
}

const agregarCarrito = (p) => {
  const existe = carrito.value.find(item => item.id === p.id)
  if (existe) existe.cantidad++
  else carrito.value.push({ ...p, cantidad: 1 })
  p.stock--
  lanzarNotificacion('🛒 Agregado al carrito')
}

const sumarStock = (p) => { p.stock++; lanzarNotificacion('➕ Stock Actualizado') }

const eliminarDelCarrito = (item) => {
  const p = productos.value.find(prod => prod.id === item.id)
  if (p) p.stock += item.cantidad
  carrito.value = carrito.value.filter(i => i.id !== item.id)
}

const finalizarCompra = () => {
  loading.value = true
  setTimeout(() => {
    const doc = new jsPDF()
    doc.text("RECIBO VUE FOOD", 80, 20)
    let y = 40
    carrito.value.forEach(i => {
      doc.text(`${i.nombre} x${i.cantidad} - ${formatearMoneda(i.precio * i.cantidad)}`, 20, y)
      y += 10
    })
    doc.text(`TOTAL: ${formatearMoneda(totalCarrito.value)}`, 20, y + 10)
    doc.output('dataurlnewwindow')
    
    carrito.value = []
    loading.value = false
    mostrarCarrito.value = false
    lanzarNotificacion('✨ ¡Compra Exitosa!')
  }, 2000)
}
</script>

<style>
/* RESET Y GENERAL */
* { box-sizing: border-box; } /* Crucial para evitar desbordes en paddings */
body { margin: 0; font-family: 'Poppins', sans-serif; background: #f4f7f6; }
.container { padding: 20px; max-width: 1400px; margin: auto; }

/* HEADER FIJO */
.header-fijo {
  position: fixed; top: 0; left: 0; right: 0;
  background: white; z-index: 1000;
  padding: 15px 25px; box-shadow: 0 2px 15px rgba(0,0,0,0.1);
}
.header-contenido { display: flex; justify-content: space-between; align-items: center; max-width: 1400px; margin: auto; }
.header-contenido h1 { margin: 0; font-size: 1.6rem; color: #ff5f6d; }
.acciones-header { display: flex; gap: 20px; align-items: center; }

.btn-toggle-form {
  background: #333; color: white; border: none; padding: 10px 18px;
  border-radius: 50px; cursor: pointer; font-weight: 600;
}

.carrito-icono { font-size: 1.8rem; cursor: pointer; position: relative; }
.badge {
  position: absolute; top: -5px; right: -8px;
  background: #ff5f6d; color: white; font-size: 0.7rem;
  padding: 2px 7px; border-radius: 50%;
}

.filtros-bar {
  display: flex; gap: 15px; margin-top: 15px; max-width: 1400px; margin-inline: auto;
}
.input-busqueda, .select-categoria {
  padding: 10px; border-radius: 10px; border: 1px solid #ddd; flex: 1;
}

.header-espaciador { height: 160px; }

/* FORMULARIO */
.formulario-container { background: white; border-radius: 20px; padding: 25px; margin-bottom: 30px; box-shadow: 0 10px 25px rgba(0,0,0,0.05); }
.form-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 15px; }
.form-grid input { padding: 12px; border: 1px solid #ddd; border-radius: 10px; }
.full-width { grid-column: 1 / -1; }
.btn-crear {
  width: 100%; margin-top: 20px; padding: 15px; background: #00c853;
  color: white; border: none; border-radius: 12px; font-weight: bold; cursor: pointer;
}

/* PRODUCTOS */
.productos-grid {
  display: grid; grid-template-columns: repeat(auto-fill, minmax(280px, 1fr)); gap: 25px;
}
.card { background: white; border-radius: 20px; overflow: hidden; box-shadow: 0 5px 15px rgba(0,0,0,0.05); transition: 0.3s; }
.card:hover { transform: translateY(-5px); }
.card-img { position: relative; height: 180px; }
.card-img img { width: 100%; height: 100%; object-fit: cover; }
.tag-categoria {
  position: absolute; top: 10px; left: 10px; background: rgba(0,0,0,0.6);
  color: white; padding: 4px 10px; border-radius: 20px; font-size: 0.7rem;
}
.card-info { padding: 15px; }
.card-info h3 { margin: 0; font-size: 1.1rem; }
.desc { color: #777; font-size: 0.85rem; margin: 8px 0; }
.precio { color: #ff5f6d; font-size: 1.2rem; margin: 10px 0; }
.stock { font-weight: bold; font-size: 0.8rem; color: #555; }
.sin-stock { color: red !important; }

.botones-card { display: flex; gap: 10px; margin-top: 15px; }
.botones-card button {
  flex: 2; padding: 10px; border: none; border-radius: 10px;
  background: #00c853; color: white; cursor: pointer; font-weight: bold;
}
.btn-stock { flex: 1 !important; background: #2196f3 !important; }
button:disabled { background: #ccc !important; cursor: not-allowed; }

/* CARRITO PANEL MODIFICADO */
.panel-carrito {
  position: fixed; right: 0; top: 0; width: 400px; height: 100%;
  background: white; z-index: 2000; display: flex; flex-direction: column;
  box-shadow: -5px 0 25px rgba(0,0,0,0.1);
}
.carrito-header { 
  padding: 20px 25px; 
  border-bottom: 1px solid #eee; 
  display: flex; 
  justify-content: space-between;
  align-items: center; 
}
.carrito-header h2 { margin: 0; }

/* Botón de cerrar estilizado de forma circular y moderna */
.cerrar-panel {
  background: #f5f5f5;
  color: #333;
  border: none;
  width: 36px;
  height: 36px;
  border-radius: 50%;
  font-size: 1.1rem;
  font-weight: bold;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s ease;
}
.cerrar-panel:hover {
  background: #ff5f6d;
  color: white;
}

.carrito-items { flex: 1; overflow-y: auto; padding: 20px 25px; }
.item-carrito {
  display: flex; justify-content: space-between; align-items: center;
  background: #f9f9f9; padding: 12px; border-radius: 12px; margin-bottom: 10px;
}
.item-total { display: flex; align-items: center; gap: 10px; font-weight: bold; }
.item-total button { background: none; border: none; cursor: pointer; }

/* Estado vacío despegado de la derecha y centrado correctamente */
.carrito-vacio {
  padding: 60px 25px;
  text-align: center;
  color: #777;
  font-size: 1.1rem;
  width: 100%;
}

.carrito-footer { padding: 25px; border-top: 1px solid #eee; }
.fila-total { display: flex; justify-content: space-between; font-size: 1.4rem; margin-bottom: 15px; }
.btn-finalizar {
  width: 100%; padding: 15px; background: #ff5f6d; color: white;
  border: none; border-radius: 12px; font-weight: bold; font-size: 1.1rem; cursor: pointer;
}

/* UI GLOBALES */
.notificacion-toast {
  position: fixed; bottom: 20px; right: 20px; background: #333;
  color: white; padding: 12px 25px; border-radius: 50px; z-index: 3000;
}
.spinner-overlay {
  position: fixed; inset: 0; background: rgba(255,255,255,0.8);
  display: flex; flex-direction: column; justify-content: center; align-items: center; z-index: 4000;
}
.spinner {
  width: 40px; height: 40px; border: 4px solid #eee;
  border-top-color: #ff5f6d; border-radius: 50%; animation: spin 1s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* TRANSICIONES */
.fade-enter-active, .fade-leave-active { transition: opacity 0.3s; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
.slide-enter-active, .slide-leave-active { transition: transform 0.4s ease; }
.slide-enter-from, .slide-leave-to { transform: translateX(100%); }

/* RESPONSIVO */
@media (max-width: 768px) {
  .panel-carrito { width: 100%; }
  .filtros-bar { flex-direction: column; }
  .header-espaciador { height: 220px; }
}
</style>
