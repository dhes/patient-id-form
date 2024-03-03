<template>
	<div id="app">
		<h1>Patient ID Form</h1>
		<form @submit.prevent="submitForm">
			<label for="patientId">Patient ID:</label>
			<input v-model="patientId" id="patientId" type="text"
				placeholder="Enter Patient ID">
			<button type="submit">Submit</button>
		</form>
		<p v-if="submitted">Submitted Patient ID: {{ patientId }}</p>
		<!-- Display the JSON response -->
		<div v-if="responseData">
			<h2>Response:</h2>
			<pre>{{ responseData }}</pre>
		</div>
		<!-- Modal -->
		<div v-if="showModal" class="modal" @click="closeModal">
			<div class="modal-content" @click.stop>
				<span class="close" @click="closeModal">&times;</span>
				<h2>Action</h2>
				<p>{{ recommendation }}</p>
				<form @submit.prevent="submitAction">
					<div v-for="(action, index) in actions" :key="index">
						<input type="radio" :id="'action' + index"
							:value="action.description" v-model="selectedAction"
							name="actions">
						<label :for="'action' + index">{{ action.description }}</label>
					</div>
					<button type="submit">Submit</button>
				</form>
			</div>
		</div>
	</div>
</template>


<script>
import axios from 'axios'; // Import Axios

export default {
	name: 'App',
	data() {
		return {
			patientId: '',
			submitted: false,
			responseData: null,
			showModal: false, // To control the visibility of the modal
			// recommendation: '', // To store the recommendation text 
			// descriptions: [], // To store descriptions from the response
			actions: [],
			selectedAction: null, // To store the selected action		
		}
	},
	methods: {
		submitForm() {
			const url = `http://localhost:8080/fhir/PlanDefinition/HIVScreening/$apply?subject=Patient/${this.patientId}`;
			axios.get(url)
				.then(response => {
					this.submitted = true;
					this.responseData = response.data;

					// Check for the nested action array and store it
					if (response.data.contained &&
						response.data.contained[0].action &&
						response.data.contained[0].action[0].action &&
						response.data.contained[0].action[0].action[0].action) {
						this.recommendation = response.data.contained[0].action[0].action[0].title;
						this.actions = response.data.contained[0].action[0].action[0].action; // Store the action array
						this.showModal = true;
					} else {
						this.actions = []; // Ensure actions is cleared if path doesn't exist
					}
				})
				.catch(error => {
					console.error("There was an error submitting the form:", error);
					this.responseData = null;
					this.actions = []; // Clear actions on error
				});
		},
		closeModal() {
			this.showModal = false;
			this.selectedAction = null; // Reset selection when modal is closed
		},
		submitAction() {
			console.log("Selected action:", this.selectedAction);
			// Here you can handle the submission of the selected action
			// For example, sending it to the server or processing it further
			this.closeModal(); // Close the modal after submission
		}
	}
}
</script>

<!-- Add your styles here -->

<style>
/* Modal styles */
.modal {
	position: fixed;
	z-index: 1;
	left: 0;
	top: 0;
	width: 100%;
	height: 100%;
	overflow: auto;
	background-color: rgba(0, 0, 0, 0.4);
}

.modal-content {
	background-color: #fefefe;
	margin: 15% auto;
	padding: 20px;
	border: 1px solid #888;
	width: 80%;
}

.close {
	color: #aaa;
	float: right;
	font-size: 28px;
	font-weight: bold;
}

.close:hover,
.close:focus {
	color: black;
	text-decoration: none;
	cursor: pointer;
}
</style>
