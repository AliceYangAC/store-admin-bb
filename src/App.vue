<template>
  <TopNav />
  <router-view
    :orders="orders"
    :products="products"
    @fetchOrders="fetchOrders"
    @completeOrder="completeOrder"
    @addProductsToList="addProductsToList"
    @updateProductInList="updateProductInList"
    @getProduct="getProduct"
    @getProducts="getProducts"
    @cancelOrder="cancelOrder"
    @shipOrder="shipOrder"
    @updateOrder="updateOrder"
  ></router-view>
</template>

<script>
import TopNav from './components/TopNav.vue';

const productServiceUrl = "/products/";
const singleProductServiceUrl = "/product/";
const makelineServiceUrl = "/makeline/";

export default {
  name: 'App',
  components: {
    TopNav
  },
  data() {
    return {
      orders: [],
      products: [],
      product: {},
    }
  },
  mounted() {
    this.getProducts();
    this.fetchOrders();
    this.polling = setInterval(() => {
      this.fetchOrders();
    }, 5000);
  },
  beforeUnmount() {
    clearInterval(this.polling);
  },
  methods: {
    resolveImageUrl(imagePath) {
      if (!imagePath || imagePath === '/placeholder.png') return '/placeholder.png';
      if (imagePath.startsWith('http')) return imagePath;

      // Use the variable we defined at the top
      let baseUrl = productServiceUrl; 
      
      // Clean up double slashes
      if (baseUrl.endsWith('/') && imagePath.startsWith('/')) {
        baseUrl = baseUrl.slice(0, -1);
      }
      return `${baseUrl}${imagePath}`;
    },
    async addProductsToList(newProduct) {
      this.products.push(newProduct);
    },
    async updateProductInList(updatedProduct) {
      const index = this.products.findIndex(product => product.id === updatedProduct.id);
      if (index !== -1) {
        this.products[index] = updatedProduct;
      }
    },
    async getProduct(id) {
      fetch(`${singleProductServiceUrl}${id}`)
        .then(response => response.json())
        .then(product => {
          this.product = { ...product };
          this.product.image = this.resolveImageUrl(product.image);
        })
        .catch(error => {
          console.log(error);
          alert('Error occurred while fetching product');
        });
    },
    async getProducts() {
      fetch(`${productServiceUrl}`) 
        .then(response => response.json())
        .then(products => {
          this.products = products.map(p => ({
            ...p,
            image: this.resolveImageUrl(p.image)
          }));
        })
        .catch(error => {
          console.log(error);
          alert('Error occurred while fetching products');
        });
    },
      async fetchOrders() {
        await fetch(`${makelineServiceUrl}order/fetch`)
        .then(response => response.json())
        .then(incomingOrders => {
          console.log(incomingOrders);
          
          if (incomingOrders) {
            // MERGE LOGIC:
            // Map over the new list from the server
            this.orders = incomingOrders.map(newOrder => {
              // Check if we already have this order in memory
            const existingOrder = this.orders.find(o => o.orderId === newOrder.orderId);
            if (existingOrder && existingOrder.deliveryTime) {
              newOrder.deliveryTime = existingOrder.deliveryTime;
            }
            
            return newOrder;
          });
        } else {
          console.log('No orders from server');
          this.orders = [];
        }
      })
      .catch(error => console.error(error));
    },
    async completeOrder(orderId) {
      await this.updateOrderStatus(orderId, 1);
      alert('Order processed successfully');
    },
    async shipOrder(orderId) {
      // update status to 2 (Shipped) 
      await this.updateOrderStatus(orderId, 2);
      alert('Order Shipped!');

      // calculate random duration
      const min = 10000;
      const max = 30000;
      const duration = Math.floor(Math.random() * (max - min + 1)) + min;

      // update local order state
      const order = this.orders.find(o => o.orderId === orderId);
      if (order) {
        order.status = 2;
        // save the duration to the order object
        order.deliveryTime = duration; 
      }

      // set timeout matching the calculated duration
      setTimeout(async () => {
        console.log(`Mocking delivery for Order ${orderId}...`);
        await this.updateOrderStatus(orderId, 3);
        
        if(order) order.status = 3; 
        
        alert(`Order ${orderId} has been Delivered!`);
      }, duration); 
    },
    async cancelOrder(orderId) {
      await fetch(`${makelineServiceUrl}order/${orderId}`, {
        method: 'DELETE',
      })
      .then(response => {
         if (response.ok) {
            this.orders = this.orders.filter(o => o.orderId !== orderId);
            // Use router to go back to list if we are on the detail page
            if (this.$route.path.includes(`/order/${orderId}`)) {
                this.$router.push('/'); 
            }
            alert('Order cancelled successfully');
         } else {
             alert('Failed to cancel order');
         }
      })
      .catch(err => console.error(err));
    },

    // 4. Update Order Items (Used when removing items)
    async updateOrder({ orderId, items }) {
      let order = this.orders.find(o => o.orderId === orderId);
      if (!order) return;
      
      // Optimistically update local state
      order.items = items;
      
      // Send full update to backend
      await fetch(`${makelineServiceUrl}order`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(order)
      })
      .then(res => {
          if(res.ok) alert('Order updated');
          else alert('Failed to update order');
      })
      .catch(err => console.error(err));
    },

    // Helper: Generic Status Updater
    async updateOrderStatus(orderId, status) {
      let order = this.orders.find(o => o.orderId === orderId);
      if (!order) return;
      
      // Create a copy with the new status
      const updatedOrder = { ...order, status: status };

      await fetch(`${makelineServiceUrl}order`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updatedOrder)
      })
      .then(response => {
        if (response.ok) {
            order.status = status; // Update local state on success
        } else {
            console.error('Failed to update status on backend');
        }
      })
      .catch(err => console.error('Failed to update status', err));
    }
  }
}
</script>

<style>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  margin-top: 120px;
  padding: 1rem;
}

footer {
  position: fixed;
  bottom: 0;
  left: 0;
  right: 0;
  background-color: #333;
  color: #fff;
  padding: 1rem;
  margin: 0;
}

nav {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

ul {
  display: flex;
  list-style: none;
  margin: 0;
  padding: 0;
}

li {
  margin: 0 1rem;
}

a {
  color: #fff;
  text-decoration: none;
}

table {
  width: 100%;
  border-collapse: collapse;
  border-spacing: 0;
}

th,
td {
  padding: 8px;
  text-align: left;
  border-bottom: 1px solid #ddd;
}

.order-detail {
  text-align: left;
}

button {
  padding: 10px;
  background-color: #005f8b;
  color: #fff;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  height: 42px;
}

button:hover {
  background-color: #005f8b;
}

.action-button {
  float: right;
}

.product-detail {
  text-align: left;
  display: flex;
  align-items: flex-start;
  justify-content: center;
  gap: 1rem;
  margin: 2rem auto;
}

.product-form {
  display: flex;
  flex-direction: column;
  align-items: left;
  justify-content: center;
  margin: 2rem auto;
  width: 50%;
}

.form-row {
  display: flex;
  flex-direction: row;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 10px;
}

.ai-button {
  margin-left: 10px;
  padding: 10px 10px;
  border-radius: 5px;
  border: none;
  background-color: #007acc;
  color: #fff;
  cursor: pointer;
}

.ai-button:hover {
  background-color: #005f8b;
}

textarea {
  width: 100%;
  padding: 5px;
  border-radius: 5px;
  border: 1px solid #ccc;
}

label {
  text-align: right;
  margin-right: 10px;
  width: 100px;
  font-weight: bold;
}

input {
  width: 100%;
  padding: 5px;
  border-radius: 5px;
  border: 1px solid #ccc;
}
</style>
