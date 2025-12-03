<template>
  <div class="order-list-container">
    
    <div class="tabs">
      <button 
        v-for="(label, status) in statusTabs" 
        :key="status"
        :class="['tab-btn', { active: currentTab === parseInt(status) }]"
        @click="currentTab = parseInt(status)"
      >
        {{ label }} ({{ getCountForStatus(status) }})
      </button>
    </div>

    <div class="table-wrapper" v-if="filteredOrders.length > 0">
      <table>
        <thead>
          <tr>
            <th></th> <th>Order ID</th>
            <th>Customer ID</th>
            <th>Total</th>
            <th>Actions</th>
          </tr>
        </thead>
        <tbody>
          <template v-for="order in filteredOrders" :key="order.orderId">
            
            <tr @click="toggleDetails(order.orderId)" class="order-row">
              <td class="expand-icon">
                {{ expandedOrderId === order.orderId ? '▼' : '▶' }}
              </td>
              <td><router-link :to="`/order/${order.orderId}`" @click.stop><span class="order-id">{{ order.orderId }}</span></router-link></td>
              <td>{{ order.customerId }}</td>
              <td>${{ orderTotal(order) }}</td>
              
              <td class="actions-cell" @click.stop>
                
                <template v-if="order.status === 0">
                  <button @click="completeOrder(order.orderId)" class="btn-small process-btn">Process</button>
                  <button @click="cancelOrder(order.orderId)" class="btn-small cancel-btn">Cancel</button>
                </template>

                <template v-else-if="order.status === 1">
                  <button @click="shipOrder(order.orderId)" class="btn-small ship-btn">Ship</button>
                  <button @click="cancelOrder(order.orderId)" class="btn-small cancel-btn">Cancel</button>
                </template>

                <template v-else-if="order.status === 2">
                  <div class="delivery-widget">
                    <div class="route-info">
                      <span class="from">Ottawa</span>
                      <span class="arrow">➜</span>
                      <span class="to">{{ order.shipping.city || 'Customer' }}</span>
                    </div>
                <div class="progress-track">
                      <div 
                        class="progress-fill" 
                        :style="{ 
                          // 1. Set the final duration (total time)
                          animationDuration: (order.totalDurationMs || 0) + 'ms',
                          // 2. Set the starting position (e.g., starts at 40% complete)
                          width: (order.progressPercent || 0) + '%', 
                          // 3. Set a delay so the animation only starts running for the remaining time
                          animationDelay: (order.totalDurationMs ? -order.totalDurationMs * (order.progressPercent / 100) : 0) + 'ms'
                        }"
                      ></div>

                      <div 
                        class="car-icon"
                        :style="{ 
                          // 1. Set the final duration (total time)
                          animationDuration: (order.totalDurationMs || 0) + 'ms',
                          // 2. Set the starting position (e.g., starts at 40% along the path)
                          left: (order.progressPercent || 0) + '%',
                          // 3. Set a delay so the animation starts running from the correct time offset
                          animationDelay: (order.totalDurationMs ? -order.totalDurationMs * (order.progressPercent / 100) : 0) + 'ms' 
                        }"
                      >
                        <svg viewBox="0 0 24 24" width="24" height="24" fill="#0046be">
                          <path d="M18.92 6.01C18.72 5.42 18.16 5 17.5 5h-11c-.66 0-1.21.42-1.42 1.01L3 12v8c0 .55.45 1 1 1h1c.55 0 1-.45 1-1v-1h12v1c0 .55.45 1 1 1h1c.55 0 1-.45 1-1v-8l-2.08-5.99zM6.5 16c-.83 0-1.5-.67-1.5-1.5S5.67 13 6.5 13s1.5.67 1.5 1.5S7.33 16 6.5 16zm11 0c-.83 0-1.5-.67-1.5-1.5s.67-1.5 1.5-1.5 1.5.67 1.5 1.5-.67 1.5-1.5 1.5zM5 11l1.5-4.5h11L19 11H5z"/>
                        </svg>
                      </div>

                    </div>
                    <!-- Display remaining duration (in seconds) -->
                    <span class="status-text">{{ (order.remainingDurationMs / 1000).toFixed(0) }}s remaining</span>
                  </div>
                </template>

                <template v-else>
                  <span class="status-done">Completed</span>
                </template>

              </td>
            </tr>

            <tr v-if="expandedOrderId === order.orderId" class="details-row">
              <td colspan="5">
                <div class="details-content">
                  <h4>Items in Order:</h4>
                  <ul>
                    <li v-for="item in order.items" :key="item.productId">
                       {{ item.quantity }}x <b>Product {{ item.productId }}</b> ({{ productLookup(item.productId) }}) - ${{ item.price }}
                    </li>
                  </ul>
                  <div class="shipping-info" v-if="order.shipping">
                    <p><b>Shipping To:</b> {{ order.shipping.address1 }}, {{ order.shipping.city }}</p>
                  </div>
                </div>
              </td>
            </tr>

          </template>
        </tbody>
      </table>
    </div>

    <div class="empty-state" v-else>
      <h3>No orders found in {{ statusTabs[currentTab] }}</h3>
    </div> 

  </div>
</template>

<script>
  export default {
    name: 'OrderList',
    props: ['orders', 'products'],
    emits: ['fetchOrders', 'completeOrder', 'cancelOrder', 'shipOrder'], 
    data() {
      return {
        currentTab: 1,
        expandedOrderId: null,
        statusTabs: {
          0: 'Ordered',
          1: 'Processing',
          2: 'Shipped',
          3: 'Delivered'
        }
      }
    },
    computed: {
      filteredOrders() {
        return this.orders.filter(o => o.status === this.currentTab);
      }
    },
    methods: {
      fetchOrders() {
        this.$emit('fetchOrders')
      },
      getCountForStatus(status) {
        return this.orders.filter(o => o.status === parseInt(status)).length;
      },
      orderTotal(order) {
        let total = 0;
        if (order.items) {
            order.items.forEach(item => {
            total += item.price * item.quantity;
            });
        }
        return total.toFixed(2);
      },
      productLookup(id) {
        // Safety check if products list isn't loaded yet
        if (!this.products || this.products.length === 0) return 'Loading...';
        const p = this.products.find(product => product.id === id);
        return p ? p.name : 'Unknown Product';
      },
      toggleDetails(id) {
        if (this.expandedOrderId === id) {
          this.expandedOrderId = null;
        } else {
          this.expandedOrderId = id;
        }
      },
      
      // --- ACTIONS ---
      completeOrder(id) {
        this.$emit('completeOrder', id);
      },
      shipOrder(id) {
        this.$emit('shipOrder', id);
      },
      cancelOrder(id) {
        if(confirm(`Are you sure you want to cancel Order #${id}?`)) {
          this.$emit('cancelOrder', id);
        }
      },
      refundOrder(id) {
        const reason = prompt("Please enter a reason for the refund:");
        if (reason) {
          // Functionally the same as cancel, but maybe log reason
          console.log(`Refunding Order ${id}. Reason: ${reason}`);
          this.$emit('cancelOrder', id);
        }
      }
    },
    beforeMount() {
      this.fetchOrders()
    }
  }
</script>

<style scoped>
.order-list-container {
  max-width: 1000px;
  margin: 20px auto;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
  overflow: hidden;
}

/* TABS STYLING */
.tabs {
  display: flex;
  background-color: #f4f4f4;
  border-bottom: 1px solid #ddd;
}

.tab-btn {
  flex: 1;
  padding: 15px;
  border: none;
  background: none;
  cursor: pointer;
  font-weight: bold;
  color: #666;
  border-bottom: 3px solid transparent;
  transition: all 0.2s;
}

.tab-btn:hover {
  background-color: #eaeaea;
}

.tab-btn.active {
  color: #0046be;
  border-bottom: 3px solid #0046be;
  background-color: white;
}

/* TABLE STYLING */
.table-wrapper {
  padding: 0;
}

table {
  width: 100%;
  border-collapse: collapse;
}

th {
  text-align: center;
  padding: 15px;
  background-color: #fafafa;
  color: #444;
  border-bottom: 2px solid #eee;
}

td {
  text-align: center;
  padding: 15px;
  border-bottom: 1px solid #eee;
}

.order-row {
  cursor: pointer;
  transition: background-color 0.1s;
}

.order-row:hover {
  background-color: #f9f9f9;
}

.expand-icon {
  width: 30px;
  text-align: center;
  color: #999;
  font-size: 0.8rem;
}

.order-id:hover {
  text-decoration: underline;
}

/* ACTION BUTTONS */
.actions-cell {
  display: flex;
  gap: 8px;
  margin:auto;
  justify-content: center;
}

.btn-small {
  padding: 6px 12px;
  border-radius: 4px;
  font-size: 0.85rem;
  cursor: pointer;
  border: 1px solid transparent;
  font-weight: bold;
}

.process-btn { background-color: #e6f0ff; color: #0046be; border-color: #0046be; }
.process-btn:hover { background-color: #0046be; color: white; }

.ship-btn { background-color: #e6fffa; color: #008000; border-color: #008000; }
.ship-btn:hover { background-color: #008000; color: white; }

.cancel-btn { background-color: #ffe6e6; color: #cc0000; border-color: #cc0000; }
.cancel-btn:hover { background-color: #cc0000; color: white; }

.btn:hover { opacity: 0.9; }

/* .refund-btn { background-color: #fff0e6; color: #d35400; border-color: #d35400; }
.refund-btn:hover { background-color: #d35400; color: white; } */

.status-done { color: #888; font-style: italic; }

/* EXPANDED DETAILS STYLING */
.details-row {
  background-color: #fcfcfc;
}

.details-content {
  padding: 10px 20px 20px 50px; /* Indent to align with text */
  text-align: left;
  color: #555;
}

.details-content ul {
  list-style: disc;
  margin-left: 20px;
}

.empty-state {
  padding: 40px;
  text-align: center;
  color: #888;
}

a {
  color: #0046be;
  text-decoration: none;
  font-weight: bold;
}

.delivery-widget {
  width: 150px; /* Fixed width for the widget */
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.progress-track {
  position: relative;
  width: 100%;
  height: 6px;
  background-color: #e0e0e0;
  border-radius: 3px;
  margin-top: 8px; /* Make room for the car on top */
}

.progress-fill {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background-color: #ffe007;
  border-radius: 3px;
  width: 0%;
  animation-name: drive-progress;
  animation-timing-function: linear;
  animation-fill-mode: forwards;
}

.car-icon {
  position: absolute;
  top: -14px;
  left: 0;
  animation-name: drive-car;
  animation-timing-function: linear;
  animation-fill-mode: forwards;
}

.status-text {
  font-size: 0.75rem;
  color: #666;
  font-style: italic;
  text-align: right;
}

/* Add to OrderList.vue <style> */

.route-info {
  font-size: 0.75rem;
  color: #555;
  margin-bottom: 2px;
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-weight: 500;
}

.arrow {
  color: #0046be; /* Brand blue */
  font-size: 0.9rem;
  padding: 0 4px;
}

.to {
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
  max-width: 65px; /* Truncate very long city names */
  text-align: right;
}

/* --- KEYFRAMES --- */
@keyframes drive-progress {
  0% { width: 0%; }
  100% { width: 100%; }
}

@keyframes drive-car {
  0% { left: 0%; }
  100% { left: calc(100% - 5px); } 
}
</style>