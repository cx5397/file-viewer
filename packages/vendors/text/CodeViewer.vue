<template>
  <pre class="code-area">
    {{ value }}
  </pre>
</template>

<script>
export default {
  name: 'CodeEditor',
  props: {
    value: {
      type: String,
      description: '值'
    }
  }
}
</script>

<style scoped>
.code-area {
  display: block;
  margin: 20px auto;
  font-size: 14px;
  width: 1000px;
  min-height: 500px;
  background: #fff;
  word-break: break-word;
  white-space: break-spaces;
  color: #000;
  border: 1px solid #dcdcdc;
  padding: 20px;
}
</style>
