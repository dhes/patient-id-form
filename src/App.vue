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
			<button type="submit">Submit</button>
		</form>
		<h2>Screening Types:</h2>
		<ul>
			<li v-for="service in screeningServices" :key="service.id">
				{{ service.title }}
			</li>
		</ul>
		<div v-if="selectedPatientId">
			<h2>Patient Summary:</h2>
			<ul>
				<li><strong>Name:</strong> {{ patientSummary.name }}</li>
				<li><strong>Age:</strong> {{ patientSummary.age }}</li>
				<li><strong>Gender:</strong> {{ patientSummary.gender }}</li>
				<li><strong>Active?:</strong> {{ patientSummary.active }}</li>
				<li><strong>Deceased?:</strong> {{ patientSummary.deceased }}</li>
				<li v-if="conditions.length > 0"><strong>Conditions:</strong>
					<ul>
						<li v-for="condition in conditions" :key="condition">{{ condition }}
						</li>
					</ul>
				</li>
				<li v-if="completedProcedures.length > 0"><strong>Completed
						Procedures:</strong>
					<ul>
						<li v-for="procedure in completedProcedures"
							:key="procedure.display">
							{{ procedure.display }} - {{ procedure.date }}
						</li>
					</ul>
				</li>
			</ul>
		</div>
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
			conditions: [],
			completedProcedures: [],
		}
	},
	mounted() {
		this.fetchPatients();
	},
	methods: {
		formatDate(dateString) {
			if (!dateString) return 'Unknown date';
			const date = new Date(dateString);
			if (!isNaN(date.getTime())) {
				// Pad the month and day with leading zeros if necessary
				const year = date.getFullYear();
				const month = (date.getMonth() + 1).toString().padStart(2, '0');
				const day = date.getDate().toString().padStart(2, '0');
				return `${year}-${month}-${day}`;
			}
			return 'Invalid date';
		},
		async fetchCompletedProcedures(patientId) {
			if (!patientId) return;
			try {
				const response = await axios.get(`http://localhost:8080/fhir/Procedure?patient=${patientId}`);
				this.completedProcedures = response.data.entry
					.filter(entry => entry.resource.status === 'completed')
					.map(entry => ({
						display: entry.resource.code.coding[0].display,
						date: this.formatDate(entry.resource.performedDateTime || entry.resource.performedPeriod?.start)
					}));
			} catch (error) {
				console.error('Error fetching patient procedures:', error);
				this.completedProcedures = []; // Reset procedures on error
			}
		},
		calculateAge(birthDate) {
			if (!birthDate) return '';
			const birthday = new Date(birthDate);
			const ageDifMs = Date.now() - birthday.getTime();
			const ageDate = new Date(ageDifMs); // Miliseconds from epoch
			return Math.abs(ageDate.getUTCFullYear() - 1970);
		},
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
					name: entry.resource.name, // Assuming the patient has a 'name' array; adjust as necessary
					birthDate: entry.resource.birthDate, // Store the birth date
					gender: entry.resource.gender,
					active: entry.resource.active,
					deceasedBoolean: entry.resource.deceasedBoolean || false, // Default to false if not provided
				}));
			} catch (error) {
				console.error('Error fetching patients:', error);
			}
		},
		async fetchPatientConditions(patientId) {
			if (!patientId) return;
			try {
				const response = await axios.get(`http://localhost:8080/fhir/Condition?patient=${patientId}`);
				this.conditions = response.data.entry.filter(entry => entry.resource.clinicalStatus &&
					entry.resource.clinicalStatus.coding.some(coding => coding.code === 'active'))
					.map(entry => entry.resource.code.coding[0].display);
			} catch (error) {
				console.error('Error fetching patient conditions:', error);
				this.conditions = []; // Reset conditions on error
			}
		},
	},
	created() {
		this.fetchScreeningServices();
	},
	computed: {
		patientSummary() {
			if (!this.selectedPatientId) return '';

			const patient = this.patients.find(p => p.id === this.selectedPatientId);
			if (!patient) return 'Patient data not available';

			const nameOutput = (() => {
				if (!patient.name || patient.name.length === 0) return 'No name available';
				const givenName = patient.name[0].given.join(" "); // Assuming multiple given names
				const familyName = patient.name[0].family;
				return `${givenName} ${familyName}`;
			})();

			const ageOutput = this.calculateAge(patient.birthDate);
			const genderOutput = patient.gender ? patient.gender : 'Unknown';
			const activeOutput = patient.active ? 'Yes' : 'No';
			const deceasedOutput = patient.deceasedBoolean ? 'Yes' : 'No';

			return {
				name: nameOutput,
				age: ageOutput,
				gender: genderOutput,
				active: activeOutput,
				deceased: deceasedOutput
			};
		},
	},
	watch: {
		selectedPatientId(newVal) {
			this.fetchPatientConditions(newVal);
			this.fetchCompletedProcedures(newVal);
		},
	},
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
