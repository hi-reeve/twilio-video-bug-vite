<script setup lang="ts">
import HelloWorld from "./components/HelloWorld.vue";
</script>

<template>
    <div>
        <suspense>
            <template #default>
                <HelloWorld />
            </template>
            <template #fallback> Loading... </template>
        </suspense>
    </div>
</template>
