<template>
	<div id="app">
		<h1>Patient ID Form</h1>
		<form @submit.prevent="submitForm">
			<label for="patientId">Patient ID:</label>
			<input v-model="patientId" id="patientId" type="text"
				placeholder="Enter Patient ID">

			<label for="screeningType">Screening Type:</label>
			<select v-model="screeningType" id="screeningType">
				<option v-for="service in screeningServices" :value="service.id" :key="service.id">
					{{ service.title }}
				</option>
			</select>

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
			screeningType: '',
			screeningServices: [], // Array to hold the fetched screening services
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
		async fetchScreeningServices() {
			try {
				const response = await axios.get('http://localhost:8080/cds-services');
				this.screeningServices = response.data.services; // Assuming the JSON structure includes a services array
			} catch (error) {
				console.error('Error fetching screening services:', error);
			}
		},
		submitForm() {
			const url = `http://localhost:8080/fhir/PlanDefinition/${this.screeningType}/$apply?subject=Patient/${this.patientId}`;
			axios.get(url)
				.then(response => {
					this.submitted = true;
					this.responseData = response.data;

					// Find the RequestGroup in the contained array
					const requestGroup = response.data.contained.find(containedItem => containedItem.resourceType === 'RequestGroup');
					// const serviceRequest = response.data.contained.find(item => item.resourceType === 'ServiceRequest');

					if (requestGroup &&
						requestGroup.action &&
						requestGroup.action.length > 0 &&
						// requestGroup.action[0].action &&
						// requestGroup.action[0].action.length > 0 &&
						requestGroup.action[0].title) {
						this.recommendation = requestGroup.action[0].title;
						this.actions = requestGroup.action[0].action; // Assuming you need to work with the nested action array
						this.showModal = true;
					} else {
						// Handle the case where the structure isn't as expected
						this.recommendation = "No recommendation found.";
						this.actions = [];
						this.showModal = true;
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
			// Check if the selected action matches the first action's description
			if (this.actions.length > 0 && this.selectedAction === this.actions[0].description) {
				// Locate the ServiceRequest resource as previously described
				const serviceRequest = this.responseData.contained.find(item => item.resourceType === 'ServiceRequest');

				if (serviceRequest) {
					// Construct the Bundle with the ServiceRequest
					const bundle = {
						resourceType: "Bundle",
						type: "transaction",
						entry: [{
							resource: serviceRequest,
							request: {
								method: "POST",
								url: "ServiceRequest"
							}
						}]
					};

					// Post the Bundle to the HAPI FHIR server
					axios.post('http://localhost:8080/fhir', bundle, {
						headers: {
							'Content-Type': 'application/fhir+json',
							// Authentication headers if needed
						}
					})
						.then(response => {
							console.log("Bundle posted successfully:", response.data);
							alert("Service request has been successfully submitted.");
							// Additional success handling
							this.showModal = false;
						})
						.catch(error => {
							console.error("Error posting Bundle:", error);
							alert("There was an error submitting the service request.");
							// Error handling
						});
				}
			} else {
				// Handle case where the first action is not selected
				// For example, notify the user or simply return without doing anything
				console.log("The first action was not selected. No service request submitted.");
			}
		}

	},
	created() {
		this.fetchScreeningServices();
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
