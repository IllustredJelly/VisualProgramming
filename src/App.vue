<script setup>
import Drawflow from 'drawflow'
import "drawflow/dist/drawflow.min.css"
import { shallowRef, ref, h, render, onMounted } from 'vue'
import * as components from './components/nodes.js'

const code = ref(null)
const listDiag = ref(null)
const nameLabel = ref(null)
const scriptList = ref([])
const overwriteWarn = ref(null)
const editor = shallowRef({})
const alertUsr = ref({
  error:false,
  background:'blue',colors:['rgba(83, 245, 142, 0.96)', 'rgba(72, 212, 163, 0.83)']
})
const nodeData = ref([
  {name:'Assignation', type:'assign', class:'Assign', in:2, out:2},
  {name:'Number', type:'num', class:'Value'},
  {name:'Variable call', type:'var', class:'Value'},
  {name:'Operation', type:'operations', class:'Operation', in:2},
  {name:'If-else block', type:'flowcon', class:'Conditional', in:3, out:3},
  {name:'For loop', type:'flowloop', class:'Loop', in:2, out:3},
  {name:'Print', type:'misc', class:'Misc', in:2}
])
var script;
var name;
var nodeList = [];
var tempSave = {};
var coords = {x:100, y:100}

function showAlert(text, color) {
  if (color == 'green') {
    alertUsr.value.colors = ['rgba(83, 245, 142, 0.96)', 'rgba(72, 212, 163, 0.83)']
  } else {
    alertUsr.value.colors = ['rgba(245, 83, 83, 0.96)', 'rgba(212, 72, 72, 0.83)']
  }
  alertUsr.value.text = text
  alertUsr.value.error = true
  setTimeout(() => {alertUsr.value.error = false}, 5000)
}

function editName(mode) {
  nameLabel.value.readOnly = false
}

function setName() {
  if (nameLabel.value.value.toLowerCase() != 'unsaved') {
    name = nameLabel.value.value
  }
  nameLabel.value.readOnly = true
}

function checkScrìpts() {
  console.log(name)
  for (let sc of scriptList.value) {
    console.log(sc.name)
    if (name == sc.name) {
      return true
    }
  }
  return false
}

function newScript() {
  name = null
  nameLabel.value.value = 'Unsaved'
  nodeList = []
  editor.value.clear()
  script = null
  code.value.data = ""
}

function requestExecution() {
  if (checkNodes()) {
    createScript()
    let stout = document.getElementById('stout')
    const http = new XMLHttpRequest()
    http.open('POST', 'http://localhost:8080/exec')
    http.addEventListener('loadstart', () => {
      stout.innerHTML = 'Executing...'
    })
    http.addEventListener('loadend', () => {
      if (http.response) {
        stout.innerHTML = http.response
      }
    })
    http.addEventListener('error', () => {
      stout.innerHTML = 'Server error...'
      showAlert('Server error, couldn\'t execute the script')
    })
    let wholeScript = ''
    for (let line of script) {
      wholeScript += line
    }
    http.send(JSON.stringify({data:wholeScript}))
  }
}

function getScriptList() {
  const http = new XMLHttpRequest()
  http.open('GET', 'http://localhost:8080/users/Admin')
  http.addEventListener('load', () => {
    if (scriptList.value.err) {
      showAlert('Successfully connected to the server', 'green')
      clearInterval(scriptList.value.id)
      scriptList.value = []
    }
    if (http.response != "empty") {
      scriptList.value = JSON.parse(http.response)
    } else {
      scriptList.value = []
    }
  })
  http.addEventListener('error', () => {
    showAlert('Server error, wait a moment or reload the page')
    if (scriptList.value.err == undefined) {
      scriptList.value = {
        err:true,
        id:setInterval(getScriptList, '15000')
      }
    }
  })
  http.send()
}

function loadScript(sc) {
  listDiag.value.close()
  const http = new XMLHttpRequest()
  http.open('GET', 'http://localhost:8080/users/Admin/' + sc.replaceAll(' ', '_'))
  http.addEventListener('loadstart', () => {
    showAlert('Loading your script...', 'green')
  })
  http.addEventListener('loadend', () => {
    if (http.response) {
      const resp = JSON.parse(http.response)
      name = resp.name
      nameLabel.value.value = name
      nodeList = JSON.parse(resp.nodeList.slice(0, -1))
      editor.value.import(JSON.parse(resp.drawflow.slice(0, -1)))
      script = resp.code.split("|")
      createScript(script)
      showAlert('Script successfully loaded', 'green')
    }
  })
  http.addEventListener('error', () => {
    showAlert('Server error, couldn\'t load your script')
  })
  http.send()
}

function saveScript() {
  if (checkNodes()) {
    if (name) {
      if (checkScrìpts()) {
        overwriteWarn.value.showModal()
      } else {
        createScript()
        const http = new XMLHttpRequest()
        let data = {}
        data.name = name
        http.open('POST', 'http://localhost:8080/users/Admin')
        data.list = JSON.stringify(nodeList) + '|'
        let wholeScript = ''
        for (let line of script) {
          wholeScript += line + '|'
        }
        data.script = wholeScript.slice(0, -1)
        data.nodes = JSON.stringify(editor.value.export()) + '|'
        http.addEventListener('loadstart', () => {
          showAlert('Saving the script...', 'green')
        })
        http.addEventListener('loadend', () => {
          console.log(http.response)
          showAlert('Script successfully saved', 'green')
          getScriptList()
        })
        http.addEventListener('error', () => {
          showAlert('Server error, couldn\'t save the script')
        })
        http.send(JSON.stringify(data))
      }
    } else {
      showAlert('You have to name your script!')
    }
  }
}

function overwriteScript() {
  overwriteWarn.value.close()
  createScript()
  const http = new XMLHttpRequest()
  let data = {}
  http.open('POST', 'http://localhost:8080/users/Admin/' + name.replaceAll(' ', '_'))
  data.list = JSON.stringify(nodeList) + '|'
  let wholeScript = ''
  for (let line of script) {
    wholeScript += line + '|'
  }
  data.script = wholeScript.slice(0, -1)
  data.nodes = JSON.stringify(editor.value.export()) + '|'
  console.log(data)
  http.addEventListener('loadstart', () => {
    showAlert('Overwritting script...', 'green')
  })
  http.addEventListener('loadend', () => {
    console.log(http.response)
    showAlert('Script successfully overwritten', 'green')
    getScriptList()
  })
  http.addEventListener('error', () => {
    showAlert('Server error, couldn\'t overwrite the script')
  })
  http.send(JSON.stringify(data))
}

function deleteScript(sc, i) {
  let buttons = document.getElementsByClassName('delete-script')
  if (sc == 'cancel') {
    buttons[i].style['background-color'] = ''
  } else if (buttons[i].style['background-color'] == '') {
    buttons[i].style['background-color'] = 'red'
  } else {
    const http = new XMLHttpRequest()
    http.open('GET', 'http://localhost:8080/users/Admin/' + sc.replaceAll(' ', '_') + '/delete')
    http.addEventListener('loadstart', () => {
      buttons[i].innerHTML = 'Deleting...'
    })
    http.addEventListener('loadend', () => {
      console.log(http.response)
      buttons[i].innerHTML = 'Delete'
      buttons[i].style['background-color'] = ''
      getScriptList()
    })
    http.addEventListener('error', () => {
      showAlert('Server error, couldn\'t delete the script')
    })
    http.send()
  }
}

function getRootNode() {
  let rootNode;
  for (let node of nodeList) {
    if (node.inputs.input_1 != undefined && node.inputs.input_1.connections.length == 0) {
      if (rootNode) {
        rootNode = 'err'
        break
      } else {
        rootNode = node
      }
    }
  }
  return rootNode
}

function getNodeFromId(id) {
  for (let node of nodeList) {
    if(id == node.id) {
      return node
    }
  }
}

function updateNodeData() {
  let i = 0;
  for (let node of nodeList) {
    let nodeInfo = editor.value.getNodeFromId(node.id)
    nodeInfo.flow_inputs = node.flow_inputs
    nodeInfo.flow_outputs = node.flow_outputs
    nodeList[i] = nodeInfo
    i++;
  }
}

function checkNodes() {
  if (nodeList.length) {
    updateNodeData()
    console.log(nodeList)
    let message;
    let rootCount = 0;
    for (let node of nodeList) {
      if (node.data.val == '' && node.class != 'Misc') {
        switch (node.class) {
          case 'Assign':
            message = 'There is an assignation node without name!'
            break
          case 'Value':
            message = `There is a ${node.name.toLowerCase()} node without a value!`
            break
          case 'Operation':
            message = 'You have to select a operation for all operation nodes!'
            break
          case 'Conditional':
            message = 'You have to select a condition for all if-else nodes!'
            break
        }
        showAlert(message)
        return false
      } else if (node.inputs.input_2 && node.inputs.input_2.connections.length == 0) {
        showAlert('You have unconnected nodes!')
        return false
      } else if (node.inputs.input_3 && node.inputs.input_3.connections.length == 0) {
        showAlert('You have unconnected nodes!')
        return false
      } else if (node.inputs.input_1 && node.inputs.input_1.connections.length == 0) {
        if (rootCount) {
          showAlert('Too many root nodes') // mejorar
          return false
        } else {
          rootCount++
        }
      }
    }
    return true
  } else {
    showAlert('You haven\'t created any nodes!');
    return false
  }
}

function createScript() {
  let execTree = generateExecTree()
  console.log(execTree)
  script = generateCode(execTree)
  let scriptBlob = new Blob(script, {type:"text/plain;charset=utf-8"})
  let scriptUrl = window.URL.createObjectURL(scriptBlob)
  code.value.data = scriptUrl
}

function resolveValueNodes(id) {
  let node = editor.value.getNodeFromId(id)
  let result
  switch (node.class) {
    case 'Assign':
    case 'Loop':
    case 'Value':
      result = node.data.val
      break
    case 'Operation':
      let a = resolveValueNodes(node.inputs.input_1.connections[0].node)
      let b = resolveValueNodes(node.inputs.input_2.connections[0].node)
      result = `${a} ${node.data.val} ${b}`
      break
    default:
      console.log('Unknown class')
  }
  return result
}

function generateCode(execTree, indentLevel) {
  let codeText = [];
  if (indentLevel == undefined) {
    indentLevel = 0;
  }
  let spaces = ' '.repeat(indentLevel * 4)
  for (let line of execTree) {
    if (line.hasOwnProperty('assignation')) {
      let node = editor.value.getNodeFromId(line.assignation)
      let result = resolveValueNodes(node.inputs.input_2.connections[0].node)
      codeText.push(spaces + `${node.data.val} = ${result}\n`)
    } else if (line.hasOwnProperty('if')) {
      let node = editor.value.getNodeFromId(line.id)
      let a = resolveValueNodes(node.inputs.input_2.connections[0].node)
      let b = resolveValueNodes(node.inputs.input_3.connections[0].node)
      codeText.push(spaces + `if ${a} ${node.data.val} ${b}:\n`)
      let ifBlock = generateCode(line['if'], indentLevel + 1)
      for (let indLine of ifBlock) {
        codeText.push(indLine)
      }
      if (line['else']) {
        codeText.push(spaces + 'else:\n')
        let elseBlock = generateCode(line['else'], indentLevel + 1)
        for (let indLine of elseBlock) {
          codeText.push(indLine)
        }
      }
    } else if (line.hasOwnProperty('for')) {
      let node = editor.value.getNodeFromId(line.id)
      let result = resolveValueNodes(node.inputs.input_2.connections[0].node)
      codeText.push(spaces + `for i in range(${result}):\n`)
      let forBlock = generateCode(line['for'], indentLevel + 1)
      for (let indLine of forBlock) {
        codeText.push(indLine)
      }
    } else if (line.hasOwnProperty('print')) {
      let node = editor.value.getNodeFromId(line.print)
      let result = resolveValueNodes(node.inputs.input_2.connections[0].node)
      codeText.push(spaces + `print(${result})\n`)
    }
  }
  return codeText
}

function generateExecTree(rootNode, execTree) {
  if (execTree == undefined) {
    execTree = []
  }
  if (rootNode == undefined) {
    rootNode = getRootNode()
    console.log(rootNode)
  }
  let nextNode;
  switch (rootNode.class) {
    case 'Assign':
    case 'Misc':
      execTree.push({[rootNode.name.toLowerCase()]:rootNode.id})
      break
    case 'Conditional':
      let conditional = {id:rootNode.id}
      nextNode = getNodeFromId(rootNode.outputs.output_2.connections[0].node);
      conditional['if'] = generateExecTree(nextNode, [])
      if (rootNode.outputs.output_3.connections.length > 0) {
        nextNode = getNodeFromId(rootNode.outputs.output_3.connections[0].node);
        conditional['else'] = generateExecTree(nextNode, [])
      }
      execTree.push(conditional)
      break
    case 'Loop':
      let loop = {id:rootNode.id}
      nextNode = getNodeFromId(rootNode.outputs.output_2.connections[0].node);
      loop['for'] = generateExecTree(nextNode, [])
      execTree.push(loop)
      break
   default:
      console.log('Unknown class')
  }
  if (rootNode.outputs.output_1.connections.length > 0) {
    nextNode = getNodeFromId(rootNode.outputs.output_1.connections[0].node);
    execTree = generateExecTree(nextNode, execTree)
  }
  return execTree
}

function addNode(data) {
  let vars = {}
  switch (data.class) {
    case 'Conditional':
      vars = {'val':'', 'con':''}
      break
    case 'Loop':
      vars = {'val':'i'}
      break
    default:
      vars = {'val':''}
  }
  editor.value.addNode(
    data.name,
    data.in? data.in : 0,
    data.out? data.out : 1,
    coords.x,
    coords.y,
    data.class,
    vars,
    data.type,
    'vue'
  )
}

onMounted(() => {
  getScriptList()

  // Initialices Drawflow
  let id = document.getElementById("drawflow");
  let Vue = { version: 3, h, render };
  editor.value = new Drawflow(id, Vue);

  // Registers all nodes
  for (let node of nodeData.value) {
    var comp;
    var props = {};
    switch (node.class) {
      case 'Assign':
        comp = components.assign
        break
      case 'Operation':
        comp = components.operations
        break
      case 'Value':
        comp = components.datatypes
        props = {'type':node.type}
        break
      case 'Conditional':
        comp = components.conditional
        break
      case 'Loop':
        comp = components.loop
        break
      case 'Misc':
        comp = components.misc
        break
      default:
        comp = components.datatypes
    }
    editor.value.registerNode(
      node.type,
      comp,
      props,
      {}
    )
  }

  // Defines flow inputs and outputs of new nodes and adds them to the node list
  editor.value.on('nodeCreated', (id) => {
    console.log('New node:', id);
    let node = editor.value.getNodeFromId(id)
    let flow_inputs = [];
    let flow_outputs = [];
    if (node.class != 'Operation' && node.inputs.input_1) {
      flow_inputs = ['input_1']
    }
    if (node.class == 'Conditional') {
      flow_outputs = ['output_1', 'output_2', 'output_3']
    } else if (node.class == 'Loop') {
      flow_outputs = ['output_1', 'output_2']
    } else if (node.class != 'Operation' && node.class != 'Value') {
      flow_outputs = ['output_1']
    }
    node.flow_inputs = flow_inputs
    node.flow_outputs = flow_outputs
    console.log(node)
    nodeList.push(node)
  })

  // Removes the deleted node from the node list
  editor.value.on('nodeRemoved', (id) => {
    nodeList = nodeList.filter(node => node.id!=id)
    console.log('Removed id:', id, nodeList)
  })

  // Checks if the created connection is valid
  editor.value.on('connectionCreated', (data) => {
    updateNodeData()
    let input = getNodeFromId(data.input_id);
    let output = getNodeFromId(data.output_id);
    let output_type = 'value';
    let input_type = 'value';

    for (let flow of output.flow_outputs) {
      if (data.output_class == flow) {
        output_type = 'flow'
      }
    }

    for (let flow of input.flow_inputs) {
      if (data.input_class == flow) {
        input_type = 'flow'
      }
    }

    if (input.inputs[data.input_class].connections.length > 1) {
      editor.value.removeSingleConnection(data.output_id, data.input_id, data.output_class, data.input_class)
      showAlert('That input is already occupied')
    } else if (output_type == 'value') {
      if (input_type == 'flow') {
        editor.value.removeSingleConnection(data.output_id, data.input_id, data.output_class, data.input_class)
        showAlert('You can\'t connect a value output to a flow input!')
      }
    } else if (output_type == 'flow') {
      if (input_type == 'value') {
        editor.value.removeSingleConnection(data.output_id, data.input_id, data.output_class, data.input_class)
        showAlert('You can\'t connect a flow output to a value input!')
      } else if (output.outputs[data.output_class].connections.length > 1) {
        editor.value.removeSingleConnection(data.output_id, data.input_id, data.output_class, data.input_class)
        showAlert('That flow output already has a connection')        
      }
    }
  })

  // Keeps track of the screen position
  editor.value.on('translate', (pos) => {
    coords.x = pos.x * -1 + 100
    coords.y = pos.y * -1 + 100
  })

  editor.value.start();
})

</script>

<template>
  <div class="box">

    <div v-if="alertUsr.error">
      <dialog open id="alert-box" :style="{ 'background-color': alertUsr.colors[0], 'border-color': alertUsr.colors[1] }">
        <p>{{alertUsr.text}}</p>
      </dialog>
    </div>

    <dialog ref="overwriteWarn">
      <h1>Warning!</h1>
      <p>This script already exists, do you wish to overwrite it?</p>
      <button @click="overwriteWarn.close()">Cancel</button>
      <button @click="overwriteScript()">Accept</button>
    </dialog>

    <dialog ref="listDiag" id="script-list">
      <button @click="listDiag.close()" id="close-list">Close</button>
      <div v-if="scriptList.err">
        <h1>Server error...</h1>
        <p>Could't get your scripts list</p>
      </div>
      <div v-else>
        <h1>Your scripts:</h1>
        <div v-if="scriptList.length == 0">
          <p>You don't have scripts</p>
        </div>
        <div v-else>
          <ul>
            <li v-for="(sc, i) in scriptList">
              <p>{{sc.name}}</p>
              <button @click="loadScript(sc.name)" class="load-script">Load</button>
              <button @click="deleteScript(sc.name, i)" @focusout="deleteScript('cancel', i)" class="delete-script">Delete</button>
            </li>
          </ul>
        </div>
      </div>
    </dialog>

    <input readonly id="script-name" ref="nameLabel" value="Unsaved" maxlength="20" title="Double click to edit" @dblclick="editName()" @focusout="setName()">

    <div class="left-panel">
      <h3>Nodes</h3>
      <ul>
        <li v-for="data in nodeData">
          <button @click="addNode(data)">{{data.name}}</button>
        </li>
      </ul>
      <button @click="newScript()" id="reset">New script</button>
    </div>

    <div id="drawflow"></div>

    <div class="right-panel">
      <button @click="requestExecution()" id="execute">Execute script</button>
      <div id="code">
        <object ref="code"></object>
      </div>
      <textarea id="stout" readonly>Waiting script execution...</textarea>
      <button @click="saveScript()" class="database" id="save">Save script</button>
      <button @click="listDiag.showModal()" class="database" id="load">Load script</button>
    </div>

  </div>
</template>

<style scoped>

.box {
  font-family: Arial, Helvetica, sans-serif;
  position: absolute;
  display: flex;
  height: 100%;
  width: 100%;
  left: 0px;
  top: 0px;
}

#alert-box {
  top: -15px;
  border: 3px solid;
  border-radius: 20px;
  z-index: 1;
  animation-name: slideIn, slideOut;
  animation-duration: 1s, 1s;
  animation-delay: 0s, 4s;
}

@keyframes slideIn {
  from {
    top: -90px;
  }
  to {
    top: -15px;
  }
}

@keyframes slideOut {
  from {
    top: -15px;
  }
  to {
    top: -90px;
  }
}

#script-list #close-list {
  position: relative;
  left: 146px;
}

#script-list ul {
  list-style-type: none;
  padding-left: 2px;
}

#script-list li {
  display: flex;
  align-items: center;
  height: 30px;
}

#script-list .load-script {
  margin-left: 5px;
  margin-right: 5px;
}

#script-name {
  position: absolute;
  left: 240px;
  top: 14px;
  z-index: 1;
  border: 0px;
  font-size: medium;
  background-color: transparent;
}

.left-panel,
.right-panel {
  height: 100%;
  background-color: rgba(83, 169, 245, 0.96);
}

.left-panel {
  width: 25%;
  padding-left: 15px;
  font-size: medium;
}

.left-panel h3 {
  position: relative;
  left: 64px;
  border: 2px solid black;
  width: 54px;
  padding: 10px;
  background-color: rgba(255, 255, 255, 0.70);
  font-size: large;
}

.left-panel ul {
  list-style-type: none;
  padding-left: 0px;
  margin-right: 14px;
}

.left-panel ul button {
  margin-bottom: 10px;
  width: 100%;
  height: 40px;
}

#reset {
  position: relative;
  top: 126px;
  width: 202px;
  height: 30px;
}

#drawflow {
  width: 100%;
  height: 100%;
  text-align: initial;
  border-left: 2px solid black;
  border-right: 2px solid black;
}

.right-panel {
  width: 30%;
}

.right-panel button {
  position: relative;
  height: 30px;
}

.right-panel .database {
  bottom: -44px;
  width: 120px;
}

#execute {
  top: 15px;
  right: -5px;
  width: 97%;
}

#save {
  left: 5px;
}

#load {
  right: -13px;
}

.right-panel div {
  position: relative;
  top: 30px;
}

#code {
  height: 400px;
  background-color: white;
  margin-left: 5px;
  margin-right: 5px;
}

#code object {
  font-size: 14px;
  background-color: transparent;
  width: 100%;
  height: 100%;
}

#stout {
  position: relative;
  bottom: -37px;
  margin-left: 5px;
  margin-right: 5px;
  resize: none;
  width: 93%;
  height: 91px;
}

</style>
