<template>
	<tr v-if="hasFilterRow">
		<th v-if="lineNumbers" aria-label="Empty Filter Cell"></th>
		<th v-if="selectable" aria-label="Empty Filter Cell"></th>
		<th v-for="(column, index) in columns" :key="index" v-if="!column.hidden" :class="getClasses(column)" :aria-label="`${column.label || 'Empty'} Filter Cell`">
			<slot name="column-filter" :column="column" :updateFilters="updateSlotFilter">
				<div v-if="isFilterable(column)">
					<input
						v-if="!isDropdown(column)"
						:name="getName(column)"
						type="text"
						class="vgt-input"
						:placeholder="getPlaceholder(column)"
						:value="columnFilters[fieldKey(column.field)]"
						@keyup.enter="updateFiltersOnEnter(column, $event.target.value)"
						@input="updateFiltersOnKeyup(column, $event.target.value)"
						:aria-label="column.label"
					/>

					<!-- options are a list of primitives -->
					<select v-if="isDropdownArray(column)" :name="getName(column)" class="vgt-select" :value="columnFilters[fieldKey(column.field)]" @change="updateFiltersImmediately(column.field, $event.target.value)" :aria-label="column.label">
						<option value="" key="-1">{{ getPlaceholder(column) }}</option>
						<option v-for="(option, i) in column.filterOptions.filterDropdownItems" :key="i" :value="option">
							{{ option }}
						</option>
					</select>

					<!-- options are a list of objects with text and value -->
					<select v-if="isDropdownObjects(column)" :name="getName(column)" class="vgt-select" :value="columnFilters[fieldKey(column.field)]" @change="updateFiltersImmediately(column.field, $event.target.value)" :aria-label="column.label">
						<option value="" key="-1">{{ getPlaceholder(column) }}</option>
						<option v-for="(option, i) in column.filterOptions.filterDropdownItems" :key="i" :value="option.value">{{ option.text }}</option>
					</select>
				</div>
			</slot>
		</th>
	</tr>
</template>

<script>
export default {
	name: 'VgtFilterRow',
	props: ['lineNumbers', 'columns', 'typedColumns', 'globalSearchEnabled', 'selectable', 'mode'],
	watch: {
		columns: {
			handler(newValue, oldValue) {
				this.populateInitialFilters();
			},
			deep: true,
			immediate: true,
		},
	},
	data() {
		return {
			columnFilters: {},
			timer: null,
		};
	},
	computed: {
		// to create a filter row, we need to
		// make sure that there is atleast 1 column
		// that requires filtering
		hasFilterRow() {
			// if (this.mode === 'remote' || !this.globalSearchEnabled) {
			for (let i = 0; i < this.columns.length; i++) {
				const col = this.columns[i];
				if (col.filterOptions && col.filterOptions.enabled) {
					return true;
				}
			}
			// }
			return false;
		},
	},
	methods: {
		fieldKey(field) {
			if (typeof field === 'function' && field.name) {
				return field.name;
			}
			return field;
		},

		reset(emitEvent = false) {
			this.columnFilters = {};

			if (emitEvent) {
				this.$emit('filter-changed', this.columnFilters);
			}
		},

		isFilterable(column) {
			return column.filterOptions && column.filterOptions.enabled;
		},

		isDropdown(column) {
			return this.isFilterable(column) && column.filterOptions.filterDropdownItems && column.filterOptions.filterDropdownItems.length;
		},

		isDropdownObjects(column) {
			return this.isDropdown(column) && typeof column.filterOptions.filterDropdownItems[0] === 'object';
		},

		isDropdownArray(column) {
			return this.isDropdown(column) && typeof column.filterOptions.filterDropdownItems[0] !== 'object';
		},

		getClasses(column) {
			const firstClass = 'filter-th';
			return column.filterOptions && column.filterOptions.styleClass ? [firstClass, ...column.filterOptions.styleClass.split(' ')].join(' ') : firstClass;
		},

		// get column's defined placeholder or default one
		getPlaceholder(column) {
			const placeholder = (this.isFilterable(column) && column.filterOptions.placeholder) || `Filter ${column.label}`;
			return placeholder;
		},

		getName(column) {
			return `vgt-${this.fieldKey(column.field)}`;
		},

		updateFiltersOnEnter(column, value) {
			if (this.timer) clearTimeout(this.timer);
			this.updateFiltersImmediately(column.field, value);
		},

		updateFiltersOnKeyup(column, value) {
			// if the trigger is enter, we don't filter on keyup
			if (column.filterOptions.trigger === 'enter') return;
			this.updateFilters(column, value);
		},

		updateSlotFilter(column, value) {
			let fieldToFilter = column.filterOptions.slotFilterField || column.field;
			if (typeof column.filterOptions.formatValue === 'function') {
				value = column.filterOptions.formatValue(value);
			}
			this.updateFiltersImmediately(fieldToFilter, value);
		},

		// since vue doesn't detect property addition and deletion, we
		// need to create helper function to set property etc
		updateFilters(column, value) {
			if (this.timer) clearTimeout(this.timer);
			this.timer = setTimeout(() => {
				this.updateFiltersImmediately(column.field, value);
			}, 400);
		},

		updateFiltersImmediately(field, value) {
			this.$set(this.columnFilters, this.fieldKey(field), value);
			this.$emit('filter-changed', this.columnFilters);
		},

		populateInitialFilters() {
			for (let i = 0; i < this.columns.length; i++) {
				const col = this.columns[i];
				// lets see if there are initial
				// filters supplied by user
				if (this.isFilterable(col) && typeof col.filterOptions.filterValue !== 'undefined' && col.filterOptions.filterValue !== null) {
					this.$set(this.columnFilters, this.fieldKey(col.field), col.filterOptions.filterValue);
					// this.updateFilters(col, col.filterOptions.filterValue);
					// this.$set(col.filterOptions, 'filterValue', undefined);
				}
			}
			//* lets emit event once all filters are set
			this.$emit('filter-changed', this.columnFilters);
		},
	},
};
</script>

<style scoped></style>
