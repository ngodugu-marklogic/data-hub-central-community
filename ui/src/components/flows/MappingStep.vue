<template>
	<v-container>
		<v-row>
			<div v-if="targetEntity && !(docUris && docUris.length > 0)">
				<em>Unable to find source documents using the specified collection or query.</em>
				<br><br>
				<em>Ingest some data that mapping can use as reference and/or edit the step </em>
				<br>
				<em>settings to use a source collection or query that will return some results.</em>
			</div>
		</v-row>

		<v-col>
			<v-row>
				<v-pagination
					v-if="docUris && docUris.length > 0"
					v-model="uriIndex"
					:length="docUris.length"
					:total-visible="5"
				></v-pagination>
				<v-spacer/>
				<div class="action-buttons">
					<v-btn color="primary" @click="validate" data-cy="mappingStep.testBtn">
						Test
						<v-icon right>play_arrow</v-icon>
					</v-btn>
					<v-btn color="primary" @click="onClear" data-cy="mappingStep.clearBtn">
						Clear
						<v-icon right>clear</v-icon>
					</v-btn>
					<v-btn color="primary" @click="preview" data-cy="mappingStep.previewBtn">
						Preview
						<v-icon right>visibility</v-icon>
					</v-btn>
				</div>
				<data-preview @closed="previewDoc = null" :preview="previewDoc"></data-preview>
			</v-row>
			<v-row>
				<v-simple-table
					item-key="name"
					class="elevation-1"
					dark
				>
					<thead>
						<tr>
							<th>
								<v-text-field
									v-model="propSearch"
									label="Name"
									placeholder="Filter Properties"
									hide-details
									clearable
									clear-icon="mdi-close-circle-outline"
									data-cy="mappingStep.filterProps"
								></v-text-field>
							</th>
							<th>Type</th>
							<th>XPath Expression</th>
							<th>Value</th>
						</tr>
					</thead>
					<tbody>
						<template v-for="(prop, idx) in entityProperties">
							<tr v-if="isPropRowVisible(prop)" :key="idx">
								<td class="clickable" @click="toggleProp(prop.name)">
									<div v-for="index in prop.indent" :key="index" class="indent"></div>
									<template v-if="prop.expandable">
										<v-icon v-if="expandedProps[prop.name]">arrow_drop_down</v-icon>
										<v-icon v-else>arrow_right</v-icon>
									</template>
									<div v-else class="icon-holder"></div>
									<span>{{prop.name}}</span>
								</td>
								<td>{{prop.type}}</td>
								<td>
									<v-text-field v-model="prop.mapping.sourcedFrom">
										<template v-slot:append>
											<v-menu bottom left>
												<template v-slot:activator="{ on }">
													<v-btn
														right
														icon
														data-cy="mapping.xpathButton"
														v-on="on">
														<v-icon>list</v-icon>
													</v-btn>
												</template>
												<v-card>
													<v-card-text>
														<v-treeview activatable dense item-key="xpath" :items="sampleDoc" open-all return-object hoverable @update:active="insertField($event[0], prop)" />
													</v-card-text>
												</v-card>
											</v-menu>
											<functions-menu :functions="functions" @selected="insertFunction($event, prop)"/>
										</template>
									</v-text-field>
								</td>
								<td class="value-column">
									<div class="error-msg" v-if="prop.error">{{prop.error}}</div>
									<div v-else>{{prop.value}}</div>
								</td>
							</tr>
						</template>
					</tbody>
				</v-simple-table>
			</v-row>
		</v-col>
	</v-container>
</template>

<script>
import _ from 'lodash'
import searchApi from '@/api/SearchApi'
import flowsApi from '@/api/FlowsApi'
import FunctionsMenu from '@/components/flows/FunctionsMenu'
import DataPreview from '@/components/flows/DataPreview'
import { mapState } from 'vuex'

export default {
	name: 'mapping-step',
	props: {
		flow: {type: Object},
		step: {type: Object}
	},
	data() {
		return {
			activeFields: [],
			entityProperties: [],
			expandedProps: {},
			propSearch: null,
			sampleDoc: [],
			previewDoc: null,
			docUris: [],
			mapping: {},
			uriIndex: null,
			functions: [],
			mapTestResp: {}
		}
	},
	components: {
		FunctionsMenu,
		DataPreview
	},
	computed: {
		targetEntity() {
			const targetEntityType = String(this.mapping.targetEntityType || this.mapping.targetEntity);
			const targetEntityTitle = targetEntityType.substring(targetEntityType.lastIndexOf("/") + 1)
			return this.entities[targetEntityType] || this.entities[targetEntityTitle];
		},
		sampleDocUri() {
			return this.mapping.sourceURI || (this.docUris ? this.docUris[0] : null)
		},
		mapName() {
			return (this.step && this.step.mapping) ? this.step.mapping.name : ''
		},
		validate() {
			return _.debounce(() => {
				if (!(this.mapping && this.sampleDocUri)) {
					return
				}
				flowsApi.validateMapping(this.mapping, this.sampleDocUri).then(resp => this.mapTestResp = resp)
			}, 500)
		},
		...mapState({
			entities: state => state.flows.entities
		})
	},
	mounted() {
		flowsApi.getFunctions().then(functions => {
			this.functions = functions
		})
		this.stepChanged()
	},
	methods: {
		stepChanged() {
			this.loadMapping()
			this.loadSampleDocs()
		},
		createEntityProperties() {
			if (!this.mapping.properties) {
				this.entityProperties = []
			}

			let props = []
			const getProps = (entity, mappings, values, parent, indent) => {
				entity.properties
					.slice()
					.sort((a, b) => a.name.toLowerCase().localeCompare(b.name.toLowerCase()))
					.forEach(p => {
						if (!mappings[p.name]) {
							mappings[p.name] = {sourcedFrom: ''}
						}
						const mapping = mappings[p.name]
						let newP = {
							...p,
							parentName: parent && parent.name,
							parent: parent,
							indent,
							type: p.datatype,
							mapping: mapping,
							xpath: mapping && mapping.sourcedFrom,
							value: values && values[p.name] && values[p.name].output,
							error: values && values[p.name] && values[p.name].errorMessage,
							expandable: false
						}
						props.push(newP)
						const ref = (p.$ref || (p.items && p.items.$ref))
						if (ref && ref.startsWith('#')) {
							newP.expandable = true
							const splits = ref.split('/')
							const refName = splits[splits.length - 1]
							newP.type = refName
							mapping.targetEntityType = ref
							const nextEnt = this.entities[refName]
							if (nextEnt) {
								if (!mapping.properties) {
									mapping.properties = {}
								}
								newP.children = nextEnt.properties.map(p => p.name.toLowerCase())
								getProps(nextEnt, mapping.properties, (values[newP.name] && values[newP.name].properties) || {}, newP, indent + 1)
							}
						}
					})
			}
			if (this.targetEntity) {
				let mapping = this.mapping
				const values = (this.mapTestResp && this.mapTestResp.properties) || {}
				getProps(this.targetEntity, mapping.properties, values, null, 0)
				this.entityProperties = props
			}
		},
		isPropRowVisible(prop) {
			let isVisible = true
			if (this.propSearch) {
				isVisible = isVisible && this.propsFilter(prop, this.propSearch)
			}
			return isVisible && (!prop.parentName || this.expandedProps[prop.parentName] && this.isPropRowVisible(prop.parent))
		},
		propsFilter(item, search) {
			const resp = item && (item.name.includes(search) ||
				(item.parentName && item.parentName.includes(search)) ||
				(item.children && item.children.find(c => c.includes(search))))
			return resp
		},
		toggleProp(propName) {
			this.$set(this.expandedProps, propName, !this.expandedProps[propName])
		},
		insertField(field, prop) {
			let context = ''
			if (prop.parent) {
				if (prop.parent && prop.parent.mapping && prop.parent.mapping.sourcedFrom) {
					context = prop.parent.mapping.sourcedFrom + '/'
				}
			}
			let sourcedFrom = prop.mapping.sourcedFrom || ''
			let xpath = field.xpath
			if (/(&|>|<|'|"|}|{|\s)/g.test(xpath)) {
				xpath = `*[local-name(.)='${this.escapeXML(xpath)}']`
			}
			this.$set(prop.mapping, 'sourcedFrom', sourcedFrom + xpath.replace(context, ''))
		},
		escapeXML(input = '') {
			return input
				.replace(/&/g, '&amp;')
				.replace(/</g, '&lt;')
				.replace(/>/g, '&gt;')
				.replace(/'/g, '&apos;')
				.replace(/"/g, '&quot;')
				.replace(/{/g, '&#123;')
				.replace(/}/g, '&#125;');
		},
		insertFunction(func, prop) {
			this.$set(prop.mapping, 'sourcedFrom', (prop.mapping.sourcedFrom || '') + func.signature)
		},
		onClear() {
			this.mapTestResp = {}
		},
		preview() {
			flowsApi
				.previewMapping({
					mappingName: this.mapName,
					format: this.step.options ? this.step.options.outputFormat: this.step.outputFormat,
					uri: this.sampleDocUri
				})
				.then(resp => this.previewDoc = resp)
		},
		loadMapping() {
			flowsApi.getMapping(this.mapName)
				.then(map => {
					this.mapping = map
					this.createEntityProperties()
				})
				.catch(err => {
					console.error(err)
				})
		},
		loadSampleDocs() {
			const sourceDatabase = this.step.options ? this.step.options.sourceDatabase: this.step.sourceDatabase;
			const sourceQuery = this.step.options ? this.step.options.sourceQuery: this.step.sourceQuery;
			searchApi.getResultsByQuery(sourceDatabase, sourceQuery, 20, true).then(response => {
				this.docUris = response.map(doc => doc.uri)

				if (this.uriIndex !== 1) {
					// causes loadSampleDoc to fire
					this.uriIndex = 1
				}
				else {
					this.loadSampleDoc()
				}
			})
		},
		loadSampleDoc() {
			if (this.sampleDocUri) {
				flowsApi.getSampleDoc(this.sampleDocUri, this.mapping.namespaces)
					.then(doc => this.sampleDoc = doc)
					.then(this.validate)
					.catch((err) => console.error(err))
			}
		},
		async saveMapping() {
			if (Object.keys(this.mapping).length) {
				await flowsApi.saveMapping(this.mapping)
			}
		}
	},
	watch: {
		uriIndex: function(newVal) {
			this.mapping.sourceURI = this.docUris[newVal]
			this.loadSampleDoc()
		},
		mapTestResp: 'createEntityProperties',
		mapping: {
			handler: async function() {
				this.createEntityProperties()
				await this.saveMapping()
				this.validate()
				//'createEntityProperties',
			},
			deep: true,
			immediate: true
		},
		step: 'stepChanged'
	}
}
</script>

<style lang="less" scoped>
div.target-entity-title {
	font-weight: normal;
	margin: 2px 0 29px 0;
	font-size: 20px;
}

.v-treeview,
.v-data-table,
table {
	width: 100%;
}

div.indent {
	width: 36px;
	display: inline-block;
}

div.icon-holder {
	width: 24px;
	display: inline-block;
}

td.clickable {
	cursor: pointer;
}

td.value-column {
	width: 25%;
	word-break: break-all;

	div {
		word-break: break-word;
	}
}

.error-msg {
	color: red;
	word-break: break-all;
}

.action-buttons {
	button {
		margin-right: 10px;
	}
}

/deep/ .v-treeview-node__root {
	cursor: pointer;
}
</style>
