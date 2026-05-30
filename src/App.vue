
<template>
  <div class="container">

    <header class="header-fijo">
      <div class="header-contenido">
        <h1>🍔 Restaurante Vue Food</h1>
        <div class="carrito" @click="mostrarCarrito = true">
          🛒 {{ carrito.length }}
        </div>
      </div>

      <div class="categorias-nav">
        <button 
          v-for="cat in categorias" 
          :key="cat" 
          :class="['btn-cat', { activo: categoriaSeleccionada === cat }]"
          @click="categoriaSeleccionada = cat"
        >
          {{ cat }}
        </button>
      </div>
    </header>

    <div class="header-espaciador"></div>

    <div class="productos">
      <div class="card" v-for="producto in productosFiltrados" :key="producto.id">
        <img :src="producto.imagen" :alt="producto.nombre" />

        <h3>{{ producto.nombre }}</h3>
        <p>{{ producto.descripcion }}</p>
        <h4>{{ formatearDinero(producto.precio) }}</h4>
        
        <p class="stock">Stock: {{ producto.stock }}</p>

        <div class="botones-card">
          <button @click="agregarCarrito(producto)" :disabled="producto.stock <= 0">
            {{ producto.stock <= 0 ? 'Sin Stock' : 'Agregar' }}
          </button>

          <button class="btn-stock" @click="sumarStock(producto)">
            + Stock
          </button>
        </div>
      </div>
    </div>

    <div v-if="mostrarCarrito" class="panel-carrito">
      <div class="carrito-header">
        <h2>Carrito</h2>
        <button class="cerrar-carrito" @click="mostrarCarrito = false">✕</button>
      </div>

      <div v-if="carrito.length === 0" class="carrito-vacio">
        <p>Tu carrito está vacío</p>
      </div>

      <div class="carrito-items-wrapper">
        <div v-for="item in carrito" :key="item.id" class="item">
          <span class="item-nombre">{{ item.nombre }}</span>
          <span class="item-cantidad">x{{ item.cantidad }}</span>
          <span class="item-precio">{{ formatearDinero(item.precio * item.cantidad) }}</span>
          <button class="btn-eliminar" @click="eliminar(item)">❌</button>
        </div>
      </div>

      <div class="carrito-footer">
        <h3>Total: {{ formatearDinero(total) }}</h3>
        <button class="finalizar" @click="finalizarCompra" :disabled="carrito.length === 0">
          Finalizar Compra
        </button>
      </div>
    </div>

    <div v-if="loading" class="spinner">
      <div class="loader-icon"></div>
      Procesando compra...
    </div>

    <div v-if="notificacion" class="notificacion">
      {{ notificacion }}
    </div>

  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import jsPDF from "jspdf";

const mostrarCarrito = ref(false)
const loading = ref(false)
const notificacion = ref('')
const categoriaSeleccionada = ref('Todos')

// Lista única de categorías calculada dinámicamente
const categorias = computed(() => {
  const lista = ['Todos']
  productos.value.forEach(p => {
    if (!lista.includes(p.categoria)) {
      lista.push(p.categoria)
    }
  })
  return lista
})

const productos = ref([
  { id: 1, nombre: 'Hamburguesa Clásica', descripcion: 'Carne y queso', precio: 18000, stock: 10, categoria: 'Hamburguesas', imagen: 'https://img.hogar.mapfre.es/wp-content/uploads/2018/09/hamburguesa-sencilla.jpg' },
  { id: 2, nombre: 'Hamburguesa Doble', descripcion: 'Doble carne', precio: 22000, stock: 10, categoria: 'Hamburguesas', imagen: 'https://progcarne.com/storage/app/uploads/public/608/6d1/8b0/6086d18b065a7811052900.jpg' },
  { id: 3, nombre: 'Hamburguesa BBQ', descripcion: 'Salsa BBQ', precio: 20000, stock: 10, categoria: 'Hamburguesas', imagen: 'https://www.recetasnestlecam.com/sites/default/files/srh_recipes/74e1a2dfe688f08eedf86a3711c8e4fb.png' },
  { id: 4, nombre: 'Pizza Personal', descripcion: 'Pizza pequeña', precio: 20000, stock: 10, categoria: 'Pizzas', imagen: 'https://www.tupperware.com/cdn/shop/articles/pypavrtlteochfsubmyt-412455.jpg?v=1647285652' },
  { id: 5, nombre: 'Pizza Hawaiana', descripcion: 'Jamón y piña', precio: 22000, stock: 10, categoria: 'Pizzas', imagen: 'https://irecetasfaciles.com/wp-content/uploads/2020/03/pizza-hawaiana.jpg' },
  { id: 6, nombre: 'Pizza Pepperoni', descripcion: 'Pepperoni', precio: 23000, stock: 10, categoria: 'Pizzas', imagen: 'https://ik.imagekit.io/smithfield/armour/4353bced-f940-00d0-8c6e-13a0a4a7f5c2/2ac60829-5178-4a6e-80cf-6ca43d862cee/Quick-and-Easy-Pepperoni-Pizza-700x700.jpeg?tr=w-1160,c-at_max,f-auto' },
  { id: 7, nombre: 'Papas Pequeñas', descripcion: 'Papas fritas', precio: 8000, stock: 10, categoria: 'Papas', imagen: 'https://houseofbpty.com/wp-content/uploads/2022/11/Baby-Potatoes-800x530.jpg' },
  { id: 8, nombre: 'Papas Grandes', descripcion: 'Papas fritas grandes', precio: 12000, stock: 10, categoria: 'Papas', imagen: 'https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhS280bw-krYPp2dukCuyXD2UFTjXCboDVPvpF33OPQcss6hQdXEk7OESRFk35p2sPrSZvKKTJSJN11nRxJp3U-aMsji9SNMFKMDHxcvKFoUx2MQ6xLXpbiF1wcGGmXd_TST84wxeDdZM1K/s1600/PATATAS+COCIDAS+Y+LUEGO+FRITAS+(3).jpg' },
  { id: 9, nombre: 'Salchipapa', descripcion: 'Salchicha y papas', precio: 15000, stock: 10, categoria: 'Papas', imagen: 'https://imag.bonviveur.com/emplatado-final-de-las-salchipapas.jpg' },
  { id: 10, nombre: 'Hot Dog', descripcion: 'Perro caliente', precio: 10000, stock: 10, categoria: 'Hot Dogs', imagen: 'https://egcexswrzkz.exactdn.com/wp-content/uploads/2026/04/perrito-caliente.jpg?strip=all&fit=1170%2C658' },
  { id: 11, nombre: 'Hot Dog Especial', descripcion: 'Perro especial', precio: 14000, stock: 10, categoria: 'Hot Dogs', imagen: 'https://www.lavanguardia.com/files/og_thumbnail/uploads/2020/04/20/5ea0a4775e8dd.jpeg' },
  { id: 12, nombre: 'Arepa con Carne', descripcion: 'Arepa rellena', precio: 12000, stock: 10, categoria: 'Arepas', imagen: 'https://familiakitchen.com/wp-content/uploads/2021/06/Carne-mechada-apepa.jpg' },
  { id: 13, nombre: 'Arepa con Pollo', descripcion: 'Arepa pollo', precio: 12000, stock: 10, categoria: 'Arepas', imagen: 'https://comedera.com/wp-content/uploads/sites/9/2025/10/Arepa-de-pollo-colombia.webp' },
  { id: 14, nombre: 'Sandwich Jamón', descripcion: 'Sandwich', precio: 9000, stock: 10, categoria: 'Sandwiches', imagen: 'https://comedera.com/wp-content/uploads/sites/9/2021/03/sandwich-de-jamon-y-queso.jpg' },
  { id: 15, nombre: 'Sandwich Pollo', descripcion: 'Sandwich pollo', precio: 11000, stock: 10, categoria: 'Sandwiches', imagen: 'https://imag.bonviveur.com/sandwich-de-pollo.jpg' },
  { id: 16, nombre: 'Gaseosa', descripcion: 'Bebida', precio: 5000, stock: 10, categoria: 'Bebidas', imagen: 'https://www.aguaparatuvida.com.ar/img/novedades/106.jpg' },
  { id: 17, nombre: 'Jugo Natural', descripcion: 'Jugo', precio: 6000, stock: 10, categoria: 'Bebidas', imagen: 'https://k-listo.com/wp-content/uploads/2021/11/JUGOS.jpg' },
  { id: 18, nombre: 'Malteada', descripcion: 'Malteada', precio: 9000, stock: 10, categoria: 'Bebidas', imagen: 'https://i.blogs.es/c6f09d/como-hacer-malteada-chocolate-cremosa-receta-facil-mundo/840_560.jpg' },
  { id: 19, nombre: 'Café', descripcion: 'Café caliente', precio: 4000, stock: 10, categoria: 'Bebidas', imagen: 'https://upload.wikimedia.org/wikipedia/commons/4/45/A_small_cup_of_coffee.JPG' },
  { id: 20, nombre: 'Capuccino', descripcion: 'Capuccino', precio: 7000, stock: 10, categoria: 'Bebidas', imagen: 'https://upload.wikimedia.org/wikipedia/commons/thumb/0/00/Cappuccino_PeB.jpg/1280px-Cappuccino_PeB.jpg' },
  { id: 21, nombre: 'Empanada', descripcion: 'Empanada', precio: 3000, stock: 10, categoria: 'Entradas', imagen: 'https://imag.bonviveur.com/empanadas-colombianas.jpg' },
  { id: 22, nombre: 'Pastel de Pollo', descripcion: 'Pastel', precio: 5000, stock: 10, categoria: 'Entradas', imagen: 'https://exhibirequipos.com/wp-content/uploads/2020/03/pastel-de-pollo-1.jpg' },
  { id: 23, nombre: 'Chorizo', descripcion: 'Chorizo', precio: 6000, stock: 10, categoria: 'Entradas', imagen: 'https://assets.tmecosys.com/image/upload/t_web_rdp_recipe_584x480_1_5x/img/recipe/ras/Assets/b66f872a179dea7b8d6f717b99077bf9/Derivates/5649f5b2588a2556b3e4517d13ed7a17404c5e6e.jpg' },
  { id: 24, nombre: 'Costillas BBQ', descripcion: 'Costillas', precio: 28000, stock: 10, categoria: 'Especialidades', imagen: 'https://i.blogs.es/5fc93e/1/650_1200.jpg' },
  { id: 25, nombre: 'Pollo Broaster', descripcion: 'Pollo', precio: 25000, stock: 10, categoria: 'Especialidades', imagen: 'https://imag.bonviveur.com/pollo-broaster.jpg' },
  { id: 26, nombre: 'Alitas BBQ', descripcion: 'Alitas', precio: 18000, stock: 10, categoria: 'Especialidades', imagen: 'https://www.unileverfoodsolutions.com.co/dam/global-ufs/mcos/NOLA/calcmenu/recipes/col-recipies/recetas-fruco-bbq/Alitas-BBQ-1200x709.jpg' },
  { id: 27, nombre: 'Alitas Picantes', descripcion: 'Alitas', precio: 18000, stock: 10, categoria: 'Especialidades', imagen: 'https://sandersonfarms.com/wp-content/uploads/2020/12/Sanderson-Farms_Kevin-Rathbuns-Spicy-Chicken-Wings_28218.jpg' },
  { id: 28, nombre: 'Helado', descripcion: 'Helado', precio: 6000, stock: 10, categoria: 'Postres', imagen: 'https://images.cookforyourlife.org/wp-content/uploads/2020/06/Chocolate-Whipped-Ice-Cream-shutterstock_1010248351.jpg' },
  { id: 29, nombre: 'Postre Tres Leches', descripcion: 'Postre', precio: 8000, stock: 10, categoria: 'Postres', imagen: 'https://cdn0.recetasgratis.net/es/posts/0/3/4/postre_de_tres_leches_frio_34430_600.jpg' },
  { id: 30, nombre: 'Brownie', descripcion: 'Brownie', precio: 7000, stock: 10, categoria: 'Postres', imagen: 'https://www.cocinista.es/download/bancorecursos/recetas/receta-brownies.jpg' }
])

const carrito = ref([])


const formatearDinero = (valor) => {
  return new Intl.NumberFormat('es-CO', {
    style: 'currency',
    currency: 'COP',
    minimumFractionDigits: 0
  }).format(valor)
}


const productosFiltrados = computed(() => {
  if (categoriaSeleccionada.value === 'Todos') {
    return productos.value
  }
  return productos.value.filter(p => p.categoria === categoriaSeleccionada.value)
})

const agregarCarrito = (producto) => {
  if (producto.stock <= 0) {
    lanzarNotificacion('Sin stock disponible')
    return
  }

  const existe = carrito.value.find(p => p.id === producto.id)
  if (existe) {
    existe.cantidad++
  } else {
    carrito.value.push({ ...producto, cantidad: 1 })
  }

  producto.stock--
  lanzarNotificacion('Producto agregado al carrito')
}

const sumarStock = (producto) => {
  producto.stock++
  lanzarNotificacion('Stock incrementado')
}

const eliminar = (item) => {
  const prod = productos.value.find(p => p.id === item.id)
  if (prod) {
    prod.stock += item.cantidad
  }

  carrito.value = carrito.value.filter(p => p.id !== item.id)
  lanzarNotificacion('Producto removido del carrito')
}

const total = computed(() => {
  return carrito.value.reduce((acc, item) => acc + (item.precio * item.cantidad), 0)
})

const lanzarNotificacion = (mensaje) => {
  notificacion.value = mensaje
  setTimeout(() => {
    if (notificacion.value === mensaje) notificacion.value = ''
  }, 2000)
}

const generarPDF = () => {
  const doc = new jsPDF()

  doc.setFontSize(20)
  doc.text('FACTURA DE COMPRA', 65, 20)

  doc.setFontSize(12)
  doc.text('Restaurante Vue Food', 20, 35)
  doc.text('NIT: 900123456', 20, 42)
  doc.text('San Gil - Santander', 20, 49)

  doc.line(20, 55, 190, 55)

  let y = 68
  carrito.value.forEach(item => {
    const desglose = `${item.nombre} x${item.cantidad}`
    const precioFormateado = formatearDinero(item.precio * item.cantidad)
    doc.text(desglose, 20, y)
    doc.text(precioFormateado, 150, y) 
    y += 10
  })

  doc.line(20, y, 190, y)
  y += 12

  doc.setFontSize(14)
  doc.text(`TOTAL A PAGAR: ${formatearDinero(total.value)}`, 20, y)

  y += 15
  doc.setFontSize(12)
  doc.text('¡Gracias por su compra en Vue Food!', 20, y)

  doc.output('dataurlnewwindow')
}

const finalizarCompra = () => {
  if (carrito.value.length === 0) return

  loading.value = true

  setTimeout(() => {
    loading.value = false
    mostrarCarrito.value = false
    generarPDF()
    carrito.value = []
    lanzarNotificacion('¡Compra procesada con éxito!')
  }, 1500)
}
</script>

<style>
body {
  margin: 0;
  padding: 0;
  background: linear-gradient(135deg, #fff8f0, #ffe8e8, #eef7ff);
  font-family: 'Poppins', 'Segoe UI', sans-serif;
}

.container {
  padding: 25px;
  max-width: 1400px;
  margin: auto;
}


.header-fijo {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  z-index: 100;
  background: white;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.08);
  padding: 10px 25px;
}

.header-contenido {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 10px 20px;
  border-radius: 15px;
  background: linear-gradient(135deg, #ff5f6d, #ffc371);
  color: white;
}

.header-contenido h1 {
  margin: 0;
  font-size: 1.5rem;
}

.header-espaciador {
  height: 140px; 
}

.carrito {
  background: white;
  color: #ff5f6d;
  padding: 8px 18px;
  border-radius: 50px;
  font-weight: 700;
  cursor: pointer;
  transition: transform 0.2s;
}

.carrito:hover {
  transform: scale(1.05);
}


.categorias-nav {
  display: flex;
  gap: 10px;
  margin-top: 10px;
  overflow-x: auto;
  padding-bottom: 5px;
}

.categorias-nav::-webkit-scrollbar {
  height: 4px;
}

.categorias-nav::-webkit-scrollbar-thumb {
  background: #ffc371;
  border-radius: 4px;
}

.btn-cat {
  background: #f0f0f0;
  border: none;
  padding: 8px 16px;
  border-radius: 20px;
  cursor: pointer;
  white-space: nowrap;
  font-weight: 500;
  transition: all 0.2s;
}

.btn-cat.activo, .btn-cat:hover {
  background: #ff5f6d;
  color: white;
}


.productos {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
  gap: 25px;
  margin-top: 15px;
}

.card {
  background: white;
  border-radius: 22px;
  overflow: hidden;
  box-shadow: 0 12px 30px rgba(0, 0, 0, .05);
  display: flex;
  flex-direction: column;
}

.card img {
  width: 100%;
  height: 200px;
  object-fit: cover;
}

.card h3, .card p, .card h4, .stock {
  margin: 8px 15px;
}

.card h4 {
  color: #2c3e50;
  font-size: 1.3rem;
}

.stock {
  font-weight: 700;
  color: #7f8c8d;
}

.botones-card {
  padding: 10px 15px 18px;
  display: flex;
  gap: 10px;
  margin-top: auto; 
}

.card button {
  flex: 2;
  border: none;
  padding: 12px;
  border-radius: 12px;
  background: linear-gradient(135deg, #00c853, #64dd17);
  color: white;
  cursor: pointer;
  font-weight: bold;
}

.card button:disabled {
  background: #bdc3c7 !important;
  cursor: not-allowed;
}

.btn-stock {
  flex: 1 !important;
  background: linear-gradient(135deg, #2196f3, #00b0ff) !important;
}


.panel-carrito {
  position: fixed;
  right: 0;
  top: 0;
  width: 400px;
  max-width: 100%; 
  height: 100%;
  background: white;
  box-shadow: -5px 0 25px rgba(0,0,0,0.15);
  z-index: 200;
  display: flex;
  flex-direction: column;
}

.carrito-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px;
  border-bottom: 1px solid #eee;
}

.cerrar-carrito {
  width: 40px;
  height: 40px;
  border: none;
  border-radius: 50%;
  background: #ff5f6d;
  color: white;
  cursor: pointer;
  font-size: 1rem;
}

.carrito-items-wrapper {
  flex: 1;
  overflow-y: auto;
  padding: 20px;
}

.carrito-vacio {
  text-align: center;
  color: #7f8c8d;
  padding-top: 50px;
}

.item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 12px;
  background: #f9f9f9;
  margin-bottom: 10px;
  border-radius: 12px;
  gap: 10px;
}

.item-nombre {
  flex: 2;
  font-weight: 500;
}

.item-cantidad {
  color: #7f8c8d;
}

.item-precio {
  font-weight: bold;
  color: #2c3e50;
}

.btn-eliminar {
  background: transparent;
  border: none;
  cursor: pointer;
}

.carrito-footer {
  padding: 20px;
  border-top: 1px solid #eee;
  background: #fff;
}

.finalizar {
  width: 100%;
  padding: 14px;
  border: none;
  border-radius: 14px;
  background: linear-gradient(135deg, #ff5f6d, #ff9966);
  color: white;
  cursor: pointer;
  font-size: 1.1rem;
  font-weight: bold;
}

.finalizar:disabled {
  background: #bdc3c7 !important;
  cursor: not-allowed;
}


.spinner {
  position: fixed;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  background: rgba(34, 34, 34, 0.95);
  color: white;
  padding: 25px 40px;
  border-radius: 14px;
  z-index: 300;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 10px;
}

.loader-icon {
  width: 30px;
  height: 30px;
  border: 4px solid #f3f3f3;
  border-top: 4px solid #ff5f6d;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}

.notificacion {
  position: fixed;
  bottom: 20px;
  right: 20px;
  background: #00c853;
  color: white;
  padding: 14px 22px;
  border-radius: 50px;
  z-index: 400;
  box-shadow: 0 4px 15px rgba(0,200,83,0.3);
}


@media(max-width: 600px){
  .header-espaciador {
    height: 160px;
  }
  .header-contenido h1 {
    font-size: 1.2rem;
  }
}
</style>
