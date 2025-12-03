<template>
  <div class="product-detail-container">
    
    <div v-if="showValidationErrors" class="error-banner">
      <p>Please correct the following errors:</p>
      <ul>
        <li v-for="error in validationErrors" :key="error">{{ error }}</li>
      </ul>
    </div>

    <div class="header-actions">
      <div class="product-header-info" style="flex: 1; margin-right: 20px;">
        <label class="input-label">Product Name</label>
        <input 
          id="product-name" 
          class="form-input input-title" 
          placeholder="e.g. UltraSlim X1 Laptop" 
          v-model="product.name" 
        />
        
        <div class="meta-row">
            <div class="half-width">
                <label class="input-label">Category</label>
                <input 
                  class="form-input" 
                  placeholder="Category" 
                  v-model="product.category" 
                />
            </div>
            <div class="half-width">
                <label class="input-label">Brand</label>
                <input 
                  class="form-input" 
                  placeholder="Brand" 
                  v-model="product.brand" 
                />
            </div>
        </div>
      </div>

      <div class="action-buttons">
        <button @click="saveProduct" class="btn save-btn">
          {{ product.id ? 'Update Product' : 'Create Product' }}
        </button>
      </div>
    </div>

    <div class="product-content">
      
      <div class="image-column">
        <label class="input-label">Product Image</label>
        
        <div class="image-placeholder">
          <img 
            v-if="product.image && product.image !== '/placeholder.png'" 
            :src="resolveImageUrl(product.image)" 
            alt="Product Preview" 
          />
          <div v-else class="no-image">No Image Selected</div>
          
          <div v-if="isUploading" class="upload-overlay">Uploading...</div>
        </div>

        <div class="file-upload-wrapper">
            <input type="file" @change="uploadImage" accept="image/*" class="file-input" />
            <span class="file-instruction">Click to upload new image</span>
        </div>
        <input type="hidden" v-model="product.image" />
      </div>

      <div class="info-column">
        
        <div class="info-group">
           <label class="input-label">Price ($)</label>
           <input 
             id="product-price" 
             class="form-input input-price" 
             placeholder="0.00" 
             v-model="product.price" 
             type="number" 
             step="0.01" 
           />
        </div>

        <div class="info-group">
          <label class="input-label">Description</label>
          <textarea 
            rows="8" 
            id="product-description" 
            class="form-input description-input" 
            placeholder="Enter full product description..." 
            v-model="product.description" 
          />
          <input type="hidden" id="product-id" v-model="product.id" />
        </div>

      </div>
    </div>

  </div>
</template>

<script>
  const productServiceUrl = '/product/';
  
  export default {
    name: 'ProductForm',
    props: ['products', 'resolveImageUrl'], 
    emits: ['addProductsToList','updateProductInList'],
    data() {
      return {
        product: {
          id: 0,
          name: '',
          image: '/placeholder.png',
          description: '',
          price: 0.00,
          category: '',
          brand: '' 
        },
        showValidationErrors: false,
        isUploading: false
      }
    },
    mounted() {
      if (this.$route.params.id) {
        const product = this.products.find(product => product.id == this.$route.params.id)
        if (product) {
            this.product = Object.assign({}, product);
        }
      }
    },
    computed: {
      validationErrors() {
        let errors = [];
        if (!this.product.name) errors.push('Please enter a name');
        if (!this.product.description) errors.push('Please enter a description');
        if (this.product.price <= 0) errors.push('Price must be greater than 0');
        if (!this.product.category) errors.push('Please enter a category');
        if (!this.product.brand) errors.push('Please enter a brand');
        return errors;
      }
    },
    methods: {
      async uploadImage(event) {
        const file = event.target.files[0];
        if (!file) return;

        this.isUploading = true;
        const formData = new FormData();
        formData.append('file', file);

        try {
            const response = await fetch(`${productServiceUrl}upload`, {
                method: 'POST',
                body: formData
            });
            
            if (response.ok) {
                const data = await response.json();
                this.product.image = data.image; 
            } else {
                alert('Failed to upload image');
            }
        } catch (error) {
            console.error(error);
            alert('Error uploading image');
        } finally {
            this.isUploading = false;
        }
      },
      saveProduct() {
        if (this.validationErrors.length > 0) {
          this.showValidationErrors = true;
          return;
        }

        let method = 'PUT';
        let path = this.$route.path;
        if (path.includes('add')) {
          method = 'POST';
        }

        this.product.price = parseFloat(this.product.price);

        fetch(`${productServiceUrl}`, {
          method: method,
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify(this.product)
        })
          .then(response => response.json())
          .then(product => {
            alert('Product saved successfully');           
            if (method === 'PUT') {
              this.$emit('updateProductInList', this.product);
            } else {
              this.$emit('addProductsToList', product);
            }
            this.$router.push(`/product/${product.id}`);
          })
          .catch(error => {
            console.log(error)
            alert('Error occurred while saving product')
          })
      }
    }
  }
</script>

<style scoped>
/* CONTAINER STYLES (Matches ProductDetail) */
.product-detail-container {
  text-align: left;
  max-width: 1000px;
  margin: 20px auto;
  padding: 20px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

/* ERROR BANNER */
.error-banner {
    background-color: #fff0f0;
    border: 1px solid #ffcccc;
    color: #cc0000;
    padding: 10px;
    border-radius: 5px;
    margin-bottom: 20px;
}
.error-banner ul {
    margin: 5px 0 0 20px;
    padding: 0;
}

/* HEADER SECTION */
.header-actions {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  border-bottom: 1px solid #eee;
  padding-bottom: 20px;
  margin-bottom: 30px;
}

/* CONTENT LAYOUT */
.product-content {
  display: flex;
  gap: 40px;
}

.image-column {
  flex: 0 0 40%;
}

.info-column {
  flex: 1;
}

/* FORM STYLING */
.input-label {
    display: block;
    font-weight: bold;
    color: #666;
    margin-bottom: 5px;
    font-size: 0.8rem;
    text-transform: uppercase;
}

.form-input {
    width: 100%;
    padding: 10px;
    border: 1px solid #ccc;
    border-radius: 4px;
    font-family: inherit;
    box-sizing: border-box; /* Ensures padding doesn't affect width */
    margin-bottom: 10px;
}

.form-input:focus {
    border-color: #0046be;
    outline: none;
}

/* Specific Input Styles to match Detail View Typography */
.input-title {
    font-size: 1.5rem;
    font-weight: bold;
    color: #333;
    padding: 10px 0;
    border: none;
    border-bottom: 2px solid #eee;
    background: transparent;
    margin-bottom: 15px;
}
.input-title:focus {
    border-bottom-color: #0046be;
}

.input-price {
    font-size: 1.5rem;
    font-weight: bold;
    color: #0046be;
    width: 150px;
}

.description-input {
    resize: vertical;
    line-height: 1.6;
}

.meta-row {
    display: flex;
    gap: 20px;
    margin-top: 10px;
}

.half-width {
    flex: 1;
}

/* IMAGE STYLING */
.image-placeholder {
  width: 100%;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid #eee;
  background-color: #fafafa;
  position: relative;
  min-height: 200px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.image-placeholder img {
  width: 100%;
  height: auto;
  display: block;
}

.no-image {
  color: #ccc;
  font-weight: bold;
}

.upload-overlay {
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    background: rgba(255,255,255,0.8);
    display: flex;
    justify-content: center;
    align-items: center;
    color: #0046be;
    font-weight: bold;
}

.file-upload-wrapper {
    margin-top: 10px;
    text-align: center;
}

.file-input {
    margin-top: 5px;
}

.file-instruction {
    display: block;
    font-size: 0.8rem;
    color: #888;
    margin-top: 5px;
}

/* BUTTONS */
.btn {
  padding: 10px 20px;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  border: 1px solid transparent;
  transition: opacity 0.2s;
  font-size: 1rem;
}

.save-btn {
  background-color: #0046be; 
  color: white; 
}

.save-btn:hover {
  background-color: #003da6;
}

/* RESPONSIVE */
@media (max-width: 768px) {
  .product-content {
    flex-direction: column;
  }
  
  .header-actions {
    flex-direction: column;
    gap: 15px;
  }
  
  .meta-row {
      flex-direction: column;
      gap: 10px;
  }
}
</style>