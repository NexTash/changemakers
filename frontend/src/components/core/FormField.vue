<template>
	<div v-if="props.type === 'link'">
		<span class="mb-2 block text-sm leading-4 text-gray-700">
			{{ props.label }}
		</span>
		<Autocomplete
			:value="modelValue"
			:placeholder="`Select ${props.doctype}`"
			:options="doctypeOptions"
			v-bind="$attrs"
			@change="(v) => emit('update:modelValue', v.value)"
		/>
	</div>

	<div v-else-if="props.type === 'datetime'">
		<span class="mb-2 block text-left text-sm leading-4 text-gray-700">
			{{ props.label }}
		</span>
		<input
			:value="modelValue"
			class="form-select block w-full"
			v-bind="$attrs"
			@change="(v) => emit('change', formatDateTime(v.target.value))"
			@input="(v) => emit('update:modelValue', formatDateTime(v.target.value))"
			@blur="onBlur"
			type="datetime-local"
		/>
	</div>

	<!-- Just to handle the attrs, maybe move to a separate component? -->
	<div v-bind="$attrs" v-else-if="props.type === 'geolocation'">
		<div v-if="props.modelValue">
			<span class="mb-2 block text-left text-sm leading-4 text-gray-700">
				{{ props.label }}
			</span>
			<a
				:href="`https://www.google.com/maps/search/?api=1&query=${coordinates.latitude}%2C${coordinates.longitude}`"
			>
				<div>
					<div
						class="flex w-fit items-center gap-2 rounded-lg bg-black py-2 px-5 text-white"
					>
						<FeatherIcon name="map" stroke-width="1.5" class="h-4 w-4" />
						<span class="text-sm">Open in Google Maps</span>
					</div>
				</div>
			</a>
		</div>
	</div>

	<div v-else-if="props.type === 'attach'">
		<!-- render file attach -->
		<span class="mb-2 block text-left text-sm leading-4 text-gray-700">
			{{ props.label }}
		</span>

		<div
			class="flex space-x-2"
			v-if="props.modelValue && typeof props.modelValue === 'string'"
		>
			<a :href="props.modelValue" class="text-blue-600">
				<div
					class="flex w-fit items-center gap-2 rounded-lg bg-black py-2 px-5 text-white"
				>
					<FeatherIcon name="download" class="h-4 w-4" />
					<span class="text-lg">Download Attachment</span>
				</div>
			</a>
			<Button
				class="rounded-lg text-lg"
				@click="handleAttachmentRemove"
				appearance="danger"
				>Remove</Button
			>
		</div>

		<input
			v-else
			class="text-lg text-gray-800 file:mr-2 file:rounded-lg file:border-0 file:py-2 file:px-5 file:text-base file:font-medium file:text-gray-800"
			type="file"
			accept="application/pdf"
			:value="modelValue"
			@change="handleFileSelect"
		/>
	</div>

	<Input
		v-else
		:readonly="props.readOnly"
		:type="props.type"
		:value="modelValue"
		:label="props.label"
		:options="props.options"
		@input="(v) => emit('update:modelValue', v)"
		@change="(v) => emit('change', v)"
		@blur="onBlur"
		v-bind="$attrs"
	/>
	<ErrorMessage
		v-if="validation?.meta?.touched"
		:message="validation?.errorMessage"
	/>
</template>

<script lang="ts" setup>
import { ref, computed, onMounted, inject, reactive } from "vue"
import { sessionInjectionKey } from "@/typing/InjectionKeys"
import {
	Autocomplete,
	ErrorMessage,
	createResource,
	Button,
	FeatherIcon,
	toast,
} from "frappe-ui"
const props = defineProps({
	type: String,
	modelValue: [String, Number, Boolean, Object, Array],
	config: Object,
	validation: {
		type: Object,
		default: () => ({}),
	},
	doctype: String,
	label: String,
	options: Array,
	readOnly: Boolean,
	formDocType: String,
})

const emit = defineEmits(["input", "change", "update:modelValue"])
const session = inject(sessionInjectionKey)

let currentUser = ref()
let doctypeList = ref([])

const doctypeOptions = computed(() => {
	if (props.doctype && doctypeList.value.data) {
		return doctypeList.value.data
	}

	return []
})

function formatDateTime(value) {
	const [date, time] = value.split(" ")
	const [day, month, year] = date.split("-")
	const [hours, minutes] = time.split(":")
	const formattedValue = `${day}-${month}-${year} ${hours}:${minutes}`
	return formattedValue
}

function onBlur() {
	props.validation.setTouched(true)
}

function handleGeoLocationFetchError() {
	// console.error(e)
	alert("Error getting device location!")
}

function getFormattedGeolocation(geoCoordinates: {
	latitude: number
	longitude: number
}): string {
	return `{"type":"FeatureCollection","features":[{"type":"Feature","properties":{},"geometry":{"type":"Point","coordinates":[${geoCoordinates.latitude},${geoCoordinates.longitude}]}}]}`
}

function extractCoordinatesFromString(featureString) {
	// feature string: {"type":"FeatureCollection","features":[{"type":"Feature","properties":{},"geometry":{"type":"Point","coordinates":[${geoCoordinates.latitude},${geoCoordinates.longitude}]}}]}
	const feature = JSON.parse(featureString)
	const coordinates = feature.features[0].geometry.coordinates
	return {
		latitude: coordinates[0],
		longitude: coordinates[1],
	}
}

const coordinates = computed(() => {
	if (!props.modelValue && props.type === "geolocation") {
		return {
			latitude: 0,
			longitude: 0,
		}
	}

	return extractCoordinatesFromString(props.modelValue)
})

const handleFile = (file) => {
	console.log(file)
}

onMounted(() => {
	if (props.type === "link" && props.doctype) {
		if (props.doctype === "User") {
			currentUser.value = session.user
			emit("update:modelValue", currentUser.value)
		}
		doctypeList.value = createResource({
			url: "changemakers.api.get_doctype_options",
			params: {
				doctype: props.doctype,
			},
			pageLength: 100,
			transform: (data) => {
				const titleField = data.title_field
				return data.docs.map((doc) => ({
					label: doc[titleField],
					value: doc.name,
				}))
			},
		})
		doctypeList.value.reload()
	}

	if (props.readOnly) {
	}

	if (props.type === "datetime") {
		const currentDate = new Date()
		emit("update:modelValue", currentDate.toISOString().slice(0, 16))
	}

	if (props.type === "geolocation" && !props.modelValue) {
		//fetch geolocation
		navigator.geolocation.getCurrentPosition((position) => {
			emit("update:modelValue", getFormattedGeolocation(position.coords))
		}, handleGeoLocationFetchError)
	}
})

const handleFileSelect = async (e) => {
	emit("update:modelValue", e.target.files[0])
}

const handleAttachmentRemove = (e) => {
	e.preventDefault()
	emit("update:modelValue", null)
}
</script>
