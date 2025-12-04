<template>
  <div class="product-detail-container">
    
    <div v-if="showValidationErrors" class="error-banner">
      <strong>Please correct the following:</strong>
      <ul>
        <li v-for="error in validationErrors" :key="error">{{ error }}</li>
      </ul>
    </div>

    <div class="form-header">
      <div class="title-section">
        <input 
          class="seamless-title" 
          placeholder="Enter Product Name" 
          v-model="product.name" 
        />
        
        <div class="meta-row">
          <div class="input-group">
            <span class="icon-label">📂</span>
            <input class="seamless-meta" placeholder="Category (e.g. Laptops)" v-model="product.category" />
          </div>
          <div class="input-group">
            <span class="icon-label">🏷️</span>
            <input class="seamless-meta" placeholder="Brand (e.g. Sony)" v-model="product.brand" />
          </div>
        </div>
      </div>

      <div class="header-actions">
        <button @click="saveProduct" class="btn save-btn">
          {{ product.id ? 'Save Changes' : 'Create Product' }}
        </button>
      </div>
    </div>

    <hr class="divider" />

    <div class="product-content">
      
      <div class="image-column">
        <div 
          class="image-uploader" 
          @click="triggerFileInput"
          :class="{ 'has-image': product.image && product.image !== '/placeholder.png' }"
        >
          <img 
            :src="resolveImageUrl(product)" 
            alt="Product Preview" 
            @error="handleImageError"
          />
          
          <div class="uploader-overlay">
            <div class="overlay-content">
              <span class="camera-icon">📷</span>
              <span>{{ isUploading ? 'Uploading...' : 'Click to Change Image' }}</span>
            </div>
          </div>
        </div>

        <input 
          type="file" 
          ref="fileInput" 
          @change="uploadImage" 
          accept="image/*" 
          style="display: none;" 
        />
      </div>

      <div class="info-column">
        
        <div class="field-block">
           <label>Price</label>
           <div class="price-wrapper">
             <span class="currency-symbol">$</span>
             <input 
               class="price-input" 
               placeholder="0.00" 
               v-model="product.price" 
               type="number" 
               step="0.01" 
             />
           </div>
        </div>

        <div class="field-block">
          <label>Description</label>
          <textarea 
            rows="10" 
            class="description-input" 
            placeholder="Describe the product features, specs, and benefits..." 
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
      // Trigger the hidden file input when the image div is clicked
      triggerFileInput() {
        this.$refs.fileInput.click();
      },
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
        if (!this.$route.params.id) method = 'POST';

        this.product.price = parseFloat(this.product.price);

        fetch(`${productServiceUrl}`, {
          method: method,
          headers: { 'Content-Type': 'application/json' },
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
        if (!this.product.name) errors.push('Name is required');
        if (!this.product.description) errors.push('Description is required');
        if (this.product.price <= 0) errors.push('Price must be greater than 0');
        if (!this.product.category) errors.push('Category is required');
        if (!this.product.brand) errors.push('Brand is required');
        return errors;
      }
    }
  }
</script>

<style scoped>
/* MAIN CONTAINER */
.product-detail-container {
  max-width: 900px;
  margin: 40px auto;
  padding: 40px;
  background: white;
  border-radius: 12px;
  box-shadow: 0 4px 20px rgba(0,0,0,0.08);
}

.divider {
  border: 0;
  border-top: 1px solid #eee;
  margin: 30px 0;
}

/* ERROR BANNER */
.error-banner {
    background-color: #fff5f5;
    border-left: 4px solid #cc0000;
    color: #cc0000;
    padding: 15px;
    border-radius: 4px;
    margin-bottom: 30px;
}
.error-banner ul {
    margin: 5px 0 0 20px;
    padding: 0;
}

/* 1. HEADER STYLES */
.form-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 20px;
}

.title-section {
  flex: 1;
}

/* Seamless Title Input */
.seamless-title {
  width: 100%;
  font-size: 2rem;
  font-weight: 800;
  color: #333;
  border: none;
  border-bottom: 2px solid transparent;
  padding: 5px 0;
  background: transparent;
  transition: border-color 0.2s;
  margin-bottom: 15px;
}
.seamless-title:focus {
  outline: none;
  border-bottom-color: #0046be;
}
.seamless-title::placeholder {
  color: #ccc;
}

/* Metadata Row */
.meta-row {
  display: flex;
  gap: 30px;
}

.input-group {
  display: flex;
  align-items: center;
  gap: 8px;
}

.icon-label {
  font-size: 1.2rem;
}

.seamless-meta {
  border: none;
  background: #f9f9f9;
  padding: 8px 12px;
  border-radius: 6px;
  font-size: 0.9rem;
  color: #555;
  font-weight: 600;
  width: 200px;
  transition: background 0.2s;
}
.seamless-meta:focus {
  outline: none;
  background: #eef4ff;
  color: #0046be;
}

/* 2. IMAGE UPLOADER (The "Graceful" part) */
.product-content {
  display: flex;
  gap: 50px;
}

.image-column {
  flex: 0 0 350px;
}

.image-uploader {
  width: 100%;
  aspect-ratio: 1 / 1; /* Keeps it square */
  border-radius: 12px;
  overflow: hidden;
  background-color: #f4f4f4;
  position: relative;
  cursor: pointer;
  border: 2px dashed #ddd; /* Dashed border suggests dropzone */
  transition: all 0.3s ease;
  display: flex;
  align-items: center;
  justify-content: center;
}

.image-uploader:hover {
  border-color: #0046be;
  box-shadow: 0 4px 15px rgba(0, 70, 190, 0.15);
}

.image-uploader img {
  width: 100%;
  height: 100%;
  object-fit: contain; /* Ensures image fits nicely */
  padding: 20px;
  transition: transform 0.3s ease;
}

.image-uploader.has-image img {
  padding: 0;
  object-fit: cover; /* Fills box if actual image */
}

/* Overlay that appears on hover */
.uploader-overlay {
  position: absolute;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0, 70, 190, 0.8);
  display: flex;
  align-items: center;
  justify-content: center;
  opacity: 0;
  transition: opacity 0.2s ease;
}

.image-uploader:hover .uploader-overlay {
  opacity: 1;
}

.overlay-content {
  color: white;
  text-align: center;
  font-weight: bold;
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.camera-icon {
  font-size: 2rem;
}

/* 3. RIGHT COLUMN STYLES */
.info-column {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 25px;
}

.field-block label {
  display: block;
  font-size: 0.75rem;
  text-transform: uppercase;
  letter-spacing: 1px;
  font-weight: bold;
  color: #888;
  margin-bottom: 8px;
}

/* Price Input Styling */
.price-wrapper {
  display: flex;
  align-items: center;
  border-bottom: 2px solid #eee;
  width: 150px;
  transition: border-color 0.2s;
}
.price-wrapper:focus-within {
  border-bottom-color: #0046be;
}

.currency-symbol {
  font-size: 1.5rem;
  color: #333;
  font-weight: bold;
  margin-right: 5px;
}

.price-input {
  border: none;
  font-size: 2rem;
  font-weight: bold;
  color: #0046be;
  width: 100%;
  background: transparent;
}
.price-input:focus {
  outline: none;
}

/* Description Styling */
.description-input {
  width: 100%;
  padding: 15px;
  border: 1px solid #eee;
  background-color: #fafafa;
  border-radius: 8px;
  line-height: 1.6;
  font-family: inherit;
  color: #444;
  resize: vertical;
  transition: all 0.2s;
}
.description-input:focus {
  outline: none;
  background-color: white;
  border-color: #0046be;
  box-shadow: 0 0 0 3px rgba(0, 70, 190, 0.1);
}

/* Action Button */
.save-btn {
  background-color: #0046be;
  color: white;
  padding: 12px 24px;
  font-size: 1rem;
  border-radius: 50px; /* Pill shape */
  border: none;
  cursor: pointer;
  font-weight: bold;
  box-shadow: 0 4px 10px rgba(0, 70, 190, 0.3);
  transition: transform 0.1s, box-shadow 0.2s;
}

.save-btn:hover {
  background-color: #003da6;
  transform: translateY(-2px);
  box-shadow: 0 6px 15px rgba(0, 70, 190, 0.4);
}

.save-btn:active {
  transform: translateY(0);
}

/* RESPONSIVE */
@media (max-width: 850px) {
  .product-content {
    flex-direction: column;
  }
  .image-column {
    flex: 0 0 auto;
    width: 100%;
    max-width: 400px;
    margin: 0 auto;
  }
  .form-header {
    flex-direction: column;
  }
  .header-actions {
    width: 100%;
    text-align: right;
  }
}
</style>