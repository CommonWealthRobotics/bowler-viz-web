<template>
    <div class="robotView">
        <RobotView3DScene class="scene"/>
    </div>
</template>

<script>
    import RobotView3DScene from "../components/RobotView3DScene";

    export default {
        name: "RobotView",
        components: {
            RobotView3DScene
        }
    }
</script>

<style scoped>
    .robotView {
        display: flex;
        height: 600px;
    }
</style>
