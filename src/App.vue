<template>
<div class="container">


<header class="header">
  <h1>🍔 Restaurante Vue Food</h1>

  <div class="carrito" @click="mostrarCarrito = true">
    🛒 {{ carrito.length }}
  </div>
</header>


<div class="productos">

  <div class="card" v-for="producto in productos" :key="producto.id">

    <img :src="producto.imagen" />

    <h3>{{ producto.nombre }}</h3>
    <p>{{ producto.descripcion }}</p>

    <h4>$ {{ producto.precio }}</h4>

    
    <p class="stock">Stock: {{ producto.stock }}</p>

    
    <div class="botones-card">
      <button @click="agregarCarrito(producto)" :disabled="producto.stock <= 0">
        Agregar
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

    <button class="cerrar-carrito" @click="mostrarCarrito = false">
      ✕
    </button>
  </div>

  <div v-if="carrito.length === 0">
    <p>Tu carrito está vacío</p>
  </div>

  <div v-for="item in carrito" :key="item.id" class="item">

    <span>{{ item.nombre }}</span>

    <span>x {{ item.cantidad }}</span>

    <span>$ {{ item.precio * item.cantidad }}</span>

    <button @click="eliminar(item)">
      ❌
    </button>

  </div>

  <h3>Total: $ {{ total }}</h3>

  <button class="finalizar" @click="finalizarCompra">
    Finalizar Compra
  </button>

</div>


<div v-if="loading" class="spinner">
Procesando compra...
</div>

<!-- Notificación -->
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

const productos = ref([

{id:1,nombre:'Hamburguesa Clásica',descripcion:'Carne y queso',precio:18000,stock:10,imagen:'https://img.hogar.mapfre.es/wp-content/uploads/2018/09/hamburguesa-sencilla.jpg'},
{id:2,nombre:'Hamburguesa Doble',descripcion:'Doble carne',precio:22000,stock:10,imagen:'https://progcarne.com/storage/app/uploads/public/608/6d1/8b0/6086d18b065a7811052900.jpg'},
{id:3,nombre:'Hamburguesa BBQ',descripcion:'Salsa BBQ',precio:20000,stock:10,imagen:'https://www.recetasnestlecam.com/sites/default/files/srh_recipes/74e1a2dfe688f08eedf86a3711c8e4fb.png'},
{id:4,nombre:'Pizza Personal',descripcion:'Pizza pequeña',precio:20000,stock:10,imagen:'https://www.tupperware.com/cdn/shop/articles/pypavrtlteochfsubmyt-412455.jpg?v=1647285652'},
{id:5,nombre:'Pizza Hawaiana',descripcion:'Jamón y piña',precio:22000,stock:10,imagen:'https://irecetasfaciles.com/wp-content/uploads/2020/03/pizza-hawaiana.jpg'},
{id:6,nombre:'Pizza Pepperoni',descripcion:'Pepperoni',precio:23000,stock:10,imagen:'https://ik.imagekit.io/smithfield/armour/4353bced-f940-00d0-8c6e-13a0a4a7f5c2/2ac60829-5178-4a6e-80cf-6ca43d862cee/Quick-and-Easy-Pepperoni-Pizza-700x700.jpeg?tr=w-1160,c-at_max,f-auto'},
{id:7,nombre:'Papas Pequeñas',descripcion:'Papas fritas',precio:8000,stock:10,imagen:'https://houseofbpty.com/wp-content/uploads/2022/11/Baby-Potatoes-800x530.jpg'},
{id:8,nombre:'Papas Grandes',descripcion:'Papas fritas grandes',precio:12000,stock:10,imagen:'https://blogger.googleusercontent.com/img/b/R29vZ2xl/AVvXsEhS280bw-krYPp2dukCuyXD2UFTjXCboDVPvpF33OPQcss6hQdXEk7OESRFk35p2sPrSZvKKTJSJN11nRxJp3U-aMsji9SNMFKMDHxcvKFoUx2MQ6xLXpbiF1wcGGmXd_TST84wxeDdZM1K/s1600/PATATAS+COCIDAS+Y+LUEGO+FRITAS+(3).jpg'},
{id:9,nombre:'Salchipapa',descripcion:'Salchicha y papas',precio:15000,stock:10,imagen:'https://imag.bonviveur.com/emplatado-final-de-las-salchipapas.jpg'},
{id:10,nombre:'Hot Dog',descripcion:'Perro caliente',precio:10000,stock:10,imagen:'https://egcexswrzkz.exactdn.com/wp-content/uploads/2026/04/perrito-caliente.jpg?strip=all&fit=1170%2C658'},
{id:11,nombre:'Hot Dog Especial',descripcion:'Perro especial',precio:14000,stock:10,imagen:'https://www.lavanguardia.com/files/og_thumbnail/uploads/2020/04/20/5ea0a4775e8dd.jpeg'},
{id:12,nombre:'Arepa con Carne',descripcion:'Arepa rellena',precio:12000,stock:10,imagen:'https://familiakitchen.com/wp-content/uploads/2021/06/Carne-mechada-apepa.jpg'},
{id:13,nombre:'Arepa con Pollo',descripcion:'Arepa pollo',precio:12000,stock:10,imagen:'https://comedera.com/wp-content/uploads/sites/9/2025/10/Arepa-de-pollo-colombia.webp'},
{id:14,nombre:'Sandwich Jamón',descripcion:'Sandwich',precio:9000,stock:10,imagen:'https://comedera.com/wp-content/uploads/sites/9/2021/03/sandwich-de-jamon-y-queso.jpg'},
{id:15,nombre:'Sandwich Pollo',descripcion:'Sandwich pollo',precio:11000,stock:10,imagen:'https://imag.bonviveur.com/sandwich-de-pollo.jpg'},
{id:16,nombre:'Gaseosa',descripcion:'Bebida',precio:5000,stock:10,imagen:'https://www.aguaparatuvida.com.ar/img/novedades/106.jpg'},
{id:17,nombre:'Jugo Natural',descripcion:'Jugo',precio:6000,stock:10,imagen:'https://k-listo.com/wp-content/uploads/2021/11/JUGOS.jpg'},
{id:18,nombre:'Malteada',descripcion:'Malteada',precio:9000,stock:10,imagen:'https://i.blogs.es/c6f09d/como-hacer-malteada-chocolate-cremosa-receta-facil-mundo/840_560.jpg'},
{id:19,nombre:'Café',descripcion:'Café caliente',precio:4000,stock:10,imagen:'https://upload.wikimedia.org/wikipedia/commons/4/45/A_small_cup_of_coffee.JPG'},
{id:20,nombre:'Capuccino',descripcion:'Capuccino',precio:7000,stock:10,imagen:'https://upload.wikimedia.org/wikipedia/commons/thumb/0/00/Cappuccino_PeB.jpg/1280px-Cappuccino_PeB.jpg'},
{id:21,nombre:'Empanada',descripcion:'Empanada',precio:3000,stock:10,imagen:'https://imag.bonviveur.com/empanadas-colombianas.jpg'},
{id:22,nombre:'Pastel de Pollo',descripcion:'Pastel',precio:5000,stock:10,imagen:'https://exhibirequipos.com/wp-content/uploads/2020/03/pastel-de-pollo-1.jpg'},
{id:23,nombre:'Chorizo',descripcion:'Chorizo',precio:6000,stock:10,imagen:'https://assets.tmecosys.com/image/upload/t_web_rdp_recipe_584x480_1_5x/img/recipe/ras/Assets/b66f872a179dea7b8d6f717b99077bf9/Derivates/5649f5b2588a2556b3e4517d13ed7a17404c5e6e.jpg'},
{id:24,nombre:'Costillas BBQ',descripcion:'Costillas',precio:28000,stock:10,imagen:'https://i.blogs.es/5fc93e/1/650_1200.jpg'},
{id:25,nombre:'Pollo Broaster',descripcion:'Pollo',precio:25000,stock:10,imagen:'https://imag.bonviveur.com/pollo-broaster.jpg'},
{id:26,nombre:'Alitas BBQ',descripcion:'Alitas',precio:18000,stock:10,imagen:'https://www.unileverfoodsolutions.com.co/dam/global-ufs/mcos/NOLA/calcmenu/recipes/col-recipies/recetas-fruco-bbq/Alitas-BBQ-1200x709.jpg'},
{id:27,nombre:'Alitas Picantes',descripcion:'Alitas',precio:18000,stock:10,imagen:'https://sandersonfarms.com/wp-content/uploads/2020/12/Sanderson-Farms_Kevin-Rathbuns-Spicy-Chicken-Wings_28218.jpg'},
{id:28,nombre:'Helado',descripcion:'Helado',precio:6000,stock:10,imagen:'https://images.cookforyourlife.org/wp-content/uploads/2020/06/Chocolate-Whipped-Ice-Cream-shutterstock_1010248351.jpg'},
{id:29,nombre:'Postre Tres Leches',descripcion:'Postre',precio:8000,stock:10,imagen:'https://cdn0.recetasgratis.net/es/posts/0/3/4/postre_de_tres_leches_frio_34430_600.jpg'},
{id:30,nombre:'Brownie',descripcion:'Brownie',precio:7000,stock:10,imagen:'https://www.cocinista.es/download/bancorecursos/recetas/receta-brownies.jpg'}

])

const carrito = ref([])

const agregarCarrito = (producto)=>{
  if(producto.stock <= 0){
    notificacion.value = 'Sin stock'
    return
  }

  const existe = carrito.value.find(p => p.id === producto.id)

  if(existe){
    existe.cantidad++
  }else{
    carrito.value.push({...producto,cantidad:1})
  }

  producto.stock--

  notificacion.value = 'Producto agregado'

  setTimeout(()=>{
    notificacion.value = ''
  },2000)
}

const sumarStock = (producto)=>{
  producto.stock++
  notificacion.value = 'Stock agregado'

  setTimeout(()=>{
    notificacion.value = ''
  },1500)
}

const eliminar = (item)=>{
  const prod = productos.value.find(p => p.id === item.id)

  if(prod){
    prod.stock += item.cantidad
  }

  carrito.value = carrito.value.filter(p => p.id !== item.id)

  notificacion.value = 'Producto eliminado'

  setTimeout(()=>{
    notificacion.value = ''
  },2000)
}

const total = computed(()=>{
  return carrito.value.reduce((acc,item)=>{
    return acc + item.precio * item.cantidad
  },0)
})

const generarPDF = ()=>{

  const doc = new jsPDF()

  doc.setFontSize(20)
  doc.text('FACTURA', 85, 20)

  doc.setFontSize(12)
  doc.text('Restaurante Vue Food', 20, 35)
  doc.text('NIT: 900123456', 20, 42)
  doc.text('San Gil - Santander', 20, 49)

  doc.line(20,55,190,55)

  let y = 68

  carrito.value.forEach(item=>{
    doc.text(`${item.nombre} x${item.cantidad} $${item.precio * item.cantidad}`,20,y)
    y += 10
  })

  doc.line(20,y,190,y)
  y += 12

  doc.text(`TOTAL: $${total.value}`,20,y)

  y += 15
  doc.text('Gracias por su compra',20,y)

  doc.output('dataurlnewwindow')
}

const finalizarCompra = ()=>{

  if(carrito.value.length === 0){
    notificacion.value = 'Carrito vacío'
    return
  }

  loading.value = true

  setTimeout(()=>{

    loading.value = false
    mostrarCarrito.value = false

    generarPDF()

    carrito.value = []

    notificacion.value = 'Compra exitosa'

    setTimeout(()=>{
      notificacion.value = ''
    },2000)

  },1500)
}
</script>

<style>
body{
margin:0;
padding:0;
background:linear-gradient(135deg,#fff8f0,#ffe8e8,#eef7ff);
font-family:'Poppins','Segoe UI',sans-serif;
}

.container{
padding:25px;
max-width:1400px;
margin:auto;
}

.header{
display:flex;
justify-content:space-between;
align-items:center;
padding:18px 25px;
border-radius:20px;
background:linear-gradient(135deg,#ff5f6d,#ffc371);
color:white;
}

.carrito{
background:white;
color:#ff5f6d;
padding:10px 18px;
border-radius:50px;
font-weight:700;
cursor:pointer;
}

.productos{
display:grid;
grid-template-columns:repeat(3,1fr);
gap:25px;
margin-top:30px;
}

.card{
background:white;
border-radius:22px;
overflow:hidden;
box-shadow:0 12px 30px rgba(0,0,0,.08);
}

.card img{
width:100%;
height:220px;
object-fit:cover;
}

.card h3,.card p,.card h4,.stock{
margin-left:15px;
margin-right:15px;
}

.stock{
font-weight:700;
color:#ff5f6d;
}

.botones-card{
padding:0 15px 18px;
display:flex;
gap:10px;
}

.card button{
flex:1;
border:none;
padding:12px;
border-radius:12px;
background:linear-gradient(135deg,#00c853,#64dd17);
color:white;
cursor:pointer;
}

.btn-stock{
background:linear-gradient(135deg,#2196f3,#00b0ff)!important;
}

.panel-carrito{
position:fixed;
right:0;
top:0;
width:360px;
height:100%;
background:white;
padding:25px;
overflow:auto;
}

.carrito-header{
display:flex;
justify-content:space-between;
align-items:center;
}

.cerrar-carrito{
width:42px;
height:42px;
border:none;
border-radius:50%;
background:#ff5f6d;
color:white;
cursor:pointer;
}

.item{
display:grid;
grid-template-columns:1fr auto auto auto;
gap:8px;
padding:10px;
background:#f8f8f8;
margin-bottom:10px;
border-radius:10px;
}

.finalizar{
width:100%;
padding:14px;
border:none;
border-radius:14px;
background:linear-gradient(135deg,#ff5f6d,#ff9966);
color:white;
cursor:pointer;
}

.spinner{
position:fixed;
top:50%;
left:50%;
transform:translate(-50%,-50%);
background:#222;
color:white;
padding:18px 28px;
border-radius:14px;
}

.notificacion{
position:fixed;
bottom:20px;
right:20px;
background:#00c853;
color:white;
padding:14px 22px;
border-radius:50px;
}

@media(max-width:900px){
.productos{
grid-template-columns:repeat(2,1fr);
}
}

@media(max-width:600px){
.productos{
grid-template-columns:1fr;
}

.panel-carrito{
width:100%;
}
}
</style>