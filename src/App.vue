<template>
	<div id="app">
		<h1>Patient ID Form</h1>
		<form @submit.prevent="submitForm">
			<label for="selectedPatientId">Patient ID:</label>
			<!--- input v-model="patientId" id="patientId" type="text" placeholder="Enter Patient ID" -->
			<select v-model="selectedPatientId">
				<option disabled value="">Select a patient</option>
				<option v-for="patient in patients" :key="patient.id"
					:value="patient.id">
					{{ patient.id }}
					<!-- Adjust if your patient name structure is different -->
				</option>
			</select>
			<label for="screeningType">Screening Type:</label>
			<select v-model="screeningType" id="screeningType">
				<option v-for="service in screeningServices" :value="service.id"
					:key="service.id">
					{{ service.title }}
				</option>
			</select>
			<button type="submit">Submit</button>
		</form>
		<p v-if="submitted">Submitted Patient ID: {{ selectedPatientId }}</p>
		<!-- was patientId-->
		<!-- Display the JSON response -->
		<!--
		<div v-if="responseData">
			<h2>Response:</h2>
			<pre>{{ responseData }}</pre>
		</div>
		-->
		<!-- Modal -->
		<div v-if="showModal" class="modal" @click="closeModal">
			<div class="modal-content" @click.stop>
				<span class="close" @click="closeModal">&times;</span>
				<h2>Recommendations</h2>
				<ul>
					<li v-for="recommendation in recommendations"
						:key="recommendation.screeningType">
						<strong>{{ recommendation.screeningType }}:</strong> {{
			recommendation.recommendation }}
					</li>
				</ul>
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
			patients: [],
			selectedPatientId: '',
			recommendations: [],
		}
	},
	mounted() {
		this.fetchPatients();
	}, methods: {
		async fetchScreeningServices() {
			try {
				const response = await axios.get('http://localhost:8080/cds-services');
				this.screeningServices = response.data.services; // Assuming the JSON structure includes a services array
			} catch (error) {
				console.error('Error fetching screening services:', error);
			}
		},
		async submitForm() {
			this.recommendations = []; // Reset recommendations
			for (const service of this.screeningServices) {
				const url = `http://localhost:8080/fhir/PlanDefinition/${service.id}/$apply?subject=Patient/${this.selectedPatientId}`;
				try {
					const response = await axios.get(url);
					const requestGroup = response.data.contained.find(containedItem => containedItem.resourceType === 'RequestGroup');
					if (requestGroup && requestGroup.action && requestGroup.action.length > 0 && requestGroup.action[0].title && requestGroup.action[0].title !== "No recommendation found.") {
						// Only push if there's a valid recommendation
						this.recommendations.push({
							screeningType: service.title,
							recommendation: requestGroup.action[0].title
						});
					}
					// Do not push "No recommendation found." to the list
				} catch (error) {
					console.error("Error fetching recommendation for service:", service.title, error);
					// You could handle errors differently or keep silent depending on your app's error handling strategy
				}
			}
			if (this.recommendations.length > 0) {
				this.showModal = true; // Show modal only if there are recommendations
			}
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
		},
		async fetchPatients() {
			try {
				const response = await axios.get('http://localhost:8080/fhir/Patient');
				this.patients = response.data.entry.map(entry => ({
					id: entry.resource.id,
					name: entry.resource.name // Assuming the patient has a 'name' array; adjust as necessary
				}));
			} catch (error) {
				console.error('Error fetching patients:', error);
			}
		},

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
