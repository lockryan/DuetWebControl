<style>
.no-preview3D-margin {
	margin-left: 0 !important;
	margin-right: 0 !important;
}
.no-preview3D-margin > .card {
	margin-left: 0 !important;
	margin-right: 0 !important;
}
.no-preview3D-margin .preview3D-container {
	padding: 0 !important;
}
</style>

<template>
	<v-layout column full-height>
		<v-flex d-block>
			<preview3D-panel :class="{ 'no-preview3D-margin' : $vuetify.breakpoint.xsOnly }"></preview3D-panel>
		</v-flex>
	</v-layout>
</template>
