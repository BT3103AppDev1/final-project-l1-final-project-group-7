<template>
  <div class="modal" v-if="isEditModalOpen">
    <div class="modal-content">
      <h2>Edit Gym</h2>
      <!-- Input fields for editing gym details -->
      <div class="form-group">
        <label for="gymName">Gym Name:</label>
        <input type="text" id="gymName" v-model="editingGym.gymName" />
      </div>
      <div class="form-group">
        <label for="description">Description:</label>
        <textarea id="description" v-model="editingGym.description"></textarea>
      </div>
      <div class="form-group">
        <label for="price">Price:</label>
        <input type="number" id="price" v-model="editingGym.price" />
      </div>
      <div class="form-group">
        <label for="address">Address:</label>
        <input type="text" id="address" v-model="editingGym.address" />
      </div>
      <div class="form-group">
        <label for="postalCode">Postal Code:</label>
        <input type="text" id="postalCode" v-model="editingGym.postalCode" />
      </div>
      <div class="form-group">
        <label for="contactNumber">Contact Number:</label>
        <input
          type="tel"
          id="contactNumber"
          v-model="editingGym.contactNumber"
        />
      </div>
      <div class="form-group">
        <label for="operationalHours">Operational Hours:</label>
        <input
          type="text"
          id="operationalHours"
          v-model="editingGym.operationalHours"
        />
      </div>
      <div class="form-group">
        <label for="amenities">Amenities:</label>
        <textarea id="amenities" v-model="editingGym.amenities"></textarea>
      </div>
      <div class="form-group">
        <label for="socialMediaLinks">Social Media Links:</label>
        <input
          type="text"
          id="socialMediaLinks"
          v-model="editingGym.socialMediaLinks"
        />
      </div>

      <!-- Display existing images -->
      <div class="form-group">
        <label for="existingImages">Existing Images:</label>
        <ul>
          <li
            v-for="(uploadedImageUrl, i) in editingGym.uploadedImageUrls"
            :key="i"
          >
            <div class="image-preview">
              <img :src="uploadedImageUrl" alt="Uploaded Image Preview" />
              <button @click="removeUploadedImage(i)">Remove</button>
            </div>
          </li>
        </ul>
      </div>

      <!-- Button to add/replace an image -->
      <div class="form-group">
        <label for="newImageUrl">Add/Replace Image URL:</label>
        <input type="text" id="newImageUrl" v-model="newImageUrl" />
        <button @click="addImage">Add/Replace</button>
      </div>

      <!-- File input for adding/replacing images -->
      <div class="form-group">
        <label for="imageFiles">Add/Replace Images:</label>
        <input
          type="file"
          id="imageFiles"
          @change="uploadImageFiles"
          accept="image/*"
          multiple
        />
      </div>

      <!-- Save Changes button -->
      <button @click="saveChanges">Save Changes</button>
      <!-- Close button to close the modal -->
      <button @click="closeModal">Close</button>
    </div>
  </div>
</template>
    
    <script>
import { getFirestore, doc, updateDoc } from "firebase/firestore";
import {
  getStorage,
  ref,
  uploadBytes,
  deleteObject,
  getDownloadURL,
} from "firebase/storage";
import { getAuth, onAuthStateChanged } from "firebase/auth";

export default {
  props: {
    isEditModalOpen: Boolean,
    editingGym: Object,
    gyms: Array,
  },
  data() {
    return {
      newImageUrl: "",
      newImageFiles: [],
    };
  },
  methods: {
    async saveChanges() {
      const db = getFirestore();
      const gymDocRef = doc(db, "gyms", this.editingGym.id);
      // Update the Gym details
      try {
        await updateDoc(gymDocRef, {
          gymName: this.editingGym.gymName,
          description: this.editingGym.description,
          address: this.editingGym.address,
          price: this.editingGym.price,
          postalCode: this.editingGym.postalCode,
          contactNumber: this.editingGym.contactNumber,
          operationalHours: this.editingGym.operationalHours,
          amenities: this.editingGym.amenities,
          socialMediaLinks: this.editingGym.socialMediaLinks,
          uploadedImageUrls: this.editingGym.uploadedImageUrls,
          gymModifiedDateTime: new Date(),
        });
        alert("Gym details updated successfully");

        // Update the local gym data to reflect the edited values
        const index = this.gyms.findIndex((g) => g.id === this.editingGym.id);
        if (index !== -1) {
          this.gyms[index] = { ...this.editingGym };
        }
        this.closeModal();
      } catch (error) {
        console.error("Error updating gym details:", error);
      }
    },
    addImage() {
    // Ensure there's a URL to add
    if (this.newImageUrl) {
      // Add the new image URL to the uploadedImageUrls array
      this.editingGym.uploadedImageUrls.push(this.newImageUrl);

      // Optionally, clear the input field after adding
      this.newImageUrl = '';
    }
  },
    async uploadImageFiles(event) {
      const auth = getAuth();
      const user = auth.currentUser;

      if (user) {
        const storage = getStorage();
        const imagesFolderRef = ref(storage, "gym_images");

        for (let i = 0; i < event.target.files.length; i++) {
          const file = event.target.files[i];
          const fileName = `${user.uid}_${Date.now()}_${file.name}`;
          const imageRef = ref(imagesFolderRef, fileName);

          try {
            await uploadBytes(imageRef, file);
            console.log("pushing", fileName);

            const storageRef = ref(storage, "gym_images/" + fileName);
            const downloadURL = await getDownloadURL(storageRef);
            this.editingGym.uploadedImageUrls.push(downloadURL);
          } catch (error) {
            console.error("Error uploading image:", error);
          }
        }
      } else {
        // Handle the case when the user is not authenticated (e.g., show an error or redirect to a login page)
      }
    },
    closeModal() {
      this.$emit("close");
    },
    async removeUploadedImage(index) {
      if (this.editingGym && this.editingGym.uploadedImageUrls) {
        const imageUrl = this.editingGym.uploadedImageUrls[index];

        if (imageUrl) {
          const confirmDeletion = window.confirm(
            "Are you sure you want to delete this image?"
          );

          if (confirmDeletion) {
            // Check if the URL is from Firebase Storage
            if (this.isFirebaseUrl(imageUrl)) {
              const storage = getStorage();
              const imageStorageRef = ref(storage, imageUrl);

              try {
                // Delete the object from Firebase Storage
                await deleteObject(imageStorageRef);
                console.log("Image deleted from Firebase Storage");
              } catch (error) {
                console.error(
                  "Error deleting image from Firebase Storage:",
                  error
                );
              }
            }

            // Remove the URL from the array
            this.editingGym.uploadedImageUrls.splice(index, 1);
          }
        }
      }
    },

    isFirebaseUrl(url) {
      // Define the pattern for a Firebase Storage URL
      // Adjust this pattern according to your Firebase Storage URL structure
      const firebaseUrlPattern =
        /^https:\/\/firebasestorage\.googleapis\.com\//;
      return firebaseUrlPattern.test(url);
    },
  },
};
</script>
    
  <style scoped>
/* Styles for the modal */
.modal {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000; /* Ensure it appears on top of other elements */
}

.modal-content {
  background: #fff;
  padding: 20px;
  border-radius: 5px;
  max-width: 80%; /* Adjust as needed */
  overflow-y: auto; /* Enable vertical scrolling if the content overflows */
  max-height: 80vh; /* Limit the maximum height to 80% of the viewport height */
}

.form-group {
  margin-bottom: 10px;
}

.image-url {
  display: flex;
  align-items: center;
}

.image-url button {
  margin-left: 10px;
}

.image-preview img {
  max-width: 100%; /* Adjust as needed */
  max-height: 200px; /* Adjust as needed */
}
</style>
  