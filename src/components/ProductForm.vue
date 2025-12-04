<template>
  <div class="product-detail-container">
    
    <div v-if="showValidationErrors" class="error-banner">
      <p>Please fix the following:</p>
      <ul>
        <li v-for="error in validationErrors" :key="error">{{ error }}</li>
      </ul>
    </div>

    <div class="header-actions">
      <div class="product-header-info">
        
        <div class="input-group full-width">
            <label class="input-label">Product Name</label>
            <input 
              id="product-name" 
              class="form-input input-title" 
              placeholder="e.g. UltraSlim X1 Laptop" 
              v-model="product.name" 
            />
        </div>
        
        <div class="meta-row">
            <div class="half-width">
                <label class="input-label">Category</label>
                <input 
                  class="form-input" 
                  placeholder="e.g. Computers" 
                  v-model="product.category" 
                />
            </div>
            <div class="half-width">
                <label class="input-label">Brand</label>
                <input 
                  class="form-input" 
                  placeholder="e.g. Sony" 
                  v-model="product.brand" 
                />
            </div>
        </div>
      </div>

      <div class="action-buttons">
        <button @click="saveProduct" class="btn save-btn">
          {{ product.id ? 'Save Changes' : 'Create Product' }}
        </button>
      </div>
    </div>

    <hr class="divider">

    <div class="product-content">
      
      <div class="image-column">
        <label class="input-label">Product Image</label>
        
        <div class="image-placeholder">
          <img 
            v-if="product.image && product.image !== '/placeholder.png'" 
            :src="resolveImageUrl(product)" 
            alt="Product Preview" 
            @error="handleImageError"
          />
          <div v-else class="no-image">No Image</div>
          
          <div v-if="isUploading" class="upload-overlay">Uploading...</div>
        </div>

        <div class="file-upload-wrapper">
            <label for="file-upload" class="custom-file-upload">
                <span v-if="!isUploading">Click to Change Image</span>
                <span v-else>Uploading...</span>
            </label>
            <input id="file-upload" type="file" @change="uploadImage" accept="image/*" />
        </div>
      </div>

      <div class="info-column">
        
        <div class="input-group">
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

        <div class="input-group">
          <label class="input-label">Description</label>
          <textarea 
            rows="8" 
            class="form-input description-input" 
            placeholder="Enter full product description..." 
            v-model="product.description" 
          />
        </div>
        
        <input type="hidden" v-model="product.id" />

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
          description: '',
          price: 0.00,
          category: '',
          brand: '',
          lastImageUpdate: Date.now()
        },
        showValidationErrors: false,
        isUploading: false
      }
    },
    watch: {
      products: {
        immediate: true, 
        handler() { this.initForm(); }
      },
      '$route.params.id': {
        immediate: true,
        handler() { this.initForm(); }
      }
    },
    methods: {
      handleImageError(e) {
        e.target.src = "/placeholder.png";
      },
      initForm() {
        const paramId = this.$route.params.id;
        if (paramId) {
            this.loadProductFromProps(paramId);
        } else {
            this.resetForm();
        }
      },
      resetForm() {
        this.product = {
          id: 0, name: '', description: '', price: 0.00, category: '', brand: '',
          lastImageUpdate: Date.now()
        };
        this.showValidationErrors = false;
        this.isUploading = false;
      },
      loadProductFromProps(paramId) {
        if (!this.products || this.products.length === 0) return;
        const foundProduct = this.products.find(p => p.id == paramId);
        if (foundProduct) {
           this.product = Object.assign({}, foundProduct);
           this.product.lastImageUpdate = Date.now();
        }
      },
      async uploadImage(event) {
        const file = event.target.files[0];
        if (!file) return;

        if (!this.product.id) {
            alert("Please save the product first before uploading an image.");
            return;
        }

        this.isUploading = true;
        const formData = new FormData();
        formData.append('file', file);
        formData.append('productId', this.product.id);

        try {
            const response = await fetch(`${productServiceUrl}upload`, {
                method: 'POST',
                body: formData
            });
            
            if (response.ok) {
                this.product.lastImageUpdate = Date.now();
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
        if (!this.$route.params.id) {
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
          .then(savedProduct => {
            alert('Product saved successfully');           
            
            this.product = { ...this.product, ...savedProduct };

            if (method === 'PUT') {
              this.$emit('updateProductInList', this.product);
            } else {
              this.$emit('addProductsToList', this.product);
            }
            this.$router.push(`/product/${this.product.id}`);
          })
          .catch(error => {
            console.log(error)
            alert('Error occurred while saving product')
          })
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
    }
  }
</script>

<style scoped>
/* MAIN CONTAINER */
.product-detail-container {
  text-align: left;
  max-width: 900px; /* Slightly tighter width for readability */
  margin: 20px auto;
  padding: 30px;
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
}

/* ERROR BANNER */
.error-banner {
    background-color: #fff0f0;
    border-left: 4px solid #cc0000;
    color: #cc0000;
    padding: 15px;
    border-radius: 4px;
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
  gap: 20px;
}

.product-header-info {
    flex: 1;
}

/* INPUT STYLING - GENERAL */
.input-group {
    margin-bottom: 15px;
}

.input-label {
    display: block;
    font-weight: 700;
    color: #888; /* Softer gray */
    margin-bottom: 4px;
    font-size: 0.75rem; /* Smaller text */
    text-transform: uppercase;
    letter-spacing: 0.5px;
}

.form-input {
    width: 100%;
    padding: 10px 12px;
    border: 1px solid #ddd;
    border-radius: 4px;
    font-family: inherit;
    font-size: 1rem;
    box-sizing: border-box; 
    transition: border-color 0.2s;
}

.form-input:focus {
    border-color: #0046be;
    outline: none;
    background-color: #f9fbff;
}

/* SPECIAL INPUT: TITLE */
/* Looks like a header, edits like an input */
.input-title {
    font-size: 1.8rem;
    font-weight: bold;
    color: #333;
    padding: 5px 0;
    border: none;
    border-bottom: 2px solid #eee; /* Only bottom border */
    background: transparent;
    border-radius: 0;
    margin-bottom: 15px;
}
.input-title:focus {
    background-color: transparent;
    border-bottom-color: #0046be;
}

/* SPECIAL INPUT: PRICE */
.input-price {
    font-size: 1.4rem;
    font-weight: bold;
    color: #0046be;
    width: 150px;
}

/* DESCRIPTION */
.description-input {
    line-height: 1.6;
    color: #444;
    resize: vertical;
}

/* META ROW (Category/Brand) */
.meta-row {
    display: flex;
    gap: 20px;
}

.half-width {
    flex: 1;
}

/* IMAGE SECTION */
.product-content {
  display: flex;
  gap: 40px;
}

.image-column {
  flex: 0 0 300px;
}

.info-column {
  flex: 1;
}

.image-placeholder {
  width: 100%;
  aspect-ratio: 1 / 1;
  border-radius: 8px;
  overflow: hidden;
  border: 1px solid #eee;
  background-color: #fafafa;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 10px;
}

.image-placeholder img {
  width: 100%;
  height: 100%;
  object-fit: contain;
  padding: 10px;
  box-sizing: border-box;
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

/* CUSTOM FILE UPLOAD LINK */
.file-upload-wrapper {
    text-align: center;
}

/* Hide the ugly standard input */
input[type="file"] {
    display: none;
}

/* Style the label to look like a link/button */
.custom-file-upload {
    display: inline-block;
    cursor: pointer;
    color: #0046be;
    font-weight: bold;
    font-size: 0.9rem;
    padding: 5px 10px;
    border-radius: 4px;
}

.custom-file-upload:hover {
    background-color: #f0f6ff;
    text-decoration: underline;
}

/* DIVIDER */
.divider {
    border: 0;
    border-top: 1px solid #eee;
    margin: 30px 0;
}

/* SAVE BUTTON */
.btn {
  padding: 10px 25px;
  border: none;
  border-radius: 4px;
  cursor: pointer;
  font-weight: bold;
  font-size: 1rem;
  transition: all 0.2s;
}

.save-btn {
  background-color: #0046be; 
  color: white; 
  box-shadow: 0 2px 5px rgba(0,0,0,0.2);
}

.save-btn:hover {
  background-color: #003da6;
  transform: translateY(-1px);
}

.action-buttons {
    /* Aligns button to top right */
    display: flex;
    align-items: flex-start;
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
  
  .image-column {
      flex: 0 0 auto;
      width: 100%;
      max-width: 400px;
      margin: 0 auto;
  }
}
</style>