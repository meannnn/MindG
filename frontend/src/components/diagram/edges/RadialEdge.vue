<script setup lang="ts">
/**
 * RadialEdge - Straight edge for radial layouts
 * Bubble map: center-to-center (circle edge to circle edge).
 * Double bubble map: topic circle edge to capsule endpoint.
 * 中间节点(similarity-*)：连左主题用左端，连右主题用右端；
 * 两侧节点(left-diff-* / right-diff-*)：左主题用右端，右主题用左端。
 */
import { computed } from 'vue'

import { type EdgeProps, useVueFlow } from '@vue-flow/core'

import { useDiagramStore } from '@/stores'

import type { MindGraphEdgeData } from '@/types'

const props = defineProps<EdgeProps<MindGraphEdgeData>>()

const { getNodes } = useVueFlow()
const diagramStore = useDiagramStore()

function isDoubleBubbleCapsule(id: string): boolean {
  return /^similarity-\d+$/.test(id) || /^left-diff-\d+$/.test(id) || /^right-diff-\d+$/.test(id)
}

function isSimilarityNode(id: string): boolean {
  return /^similarity-\d+$/.test(id)
}

// Compute path: for double_bubble_map, capsule side uses pill endpoint facing the topic
const path = computed(() => {
  const nodes = getNodes.value
  const sourceNode = nodes.find((n) => n.id === props.source)
  const targetNode = nodes.find((n) => n.id === props.target)

  if (!sourceNode || !targetNode) {
    return { edgePath: '', labelX: 0, labelY: 0 }
  }

  const isDoubleBubble = diagramStore.type === 'double_bubble_map'
  const sourceWidth = sourceNode.dimensions?.width ?? sourceNode.width ?? sourceNode.data?.style?.size ?? 120
  const sourceHeight = sourceNode.dimensions?.height ?? sourceNode.height ?? sourceNode.data?.style?.size ?? 120
  const targetWidth = targetNode.dimensions?.width ?? targetNode.width ?? targetNode.data?.style?.size ?? 80
  const targetHeight = targetNode.dimensions?.height ?? targetNode.height ?? targetNode.data?.style?.size ?? 80

  const sourceCenterX = sourceNode.position.x + Number(sourceWidth) / 2
  const sourceCenterY = sourceNode.position.y + Number(sourceHeight) / 2
  const targetCenterX = targetNode.position.x + Number(targetWidth) / 2
  const targetCenterY = targetNode.position.y + Number(targetHeight) / 2

  let startX: number
  let startY: number
  let endX: number
  let endY: number

  if (isDoubleBubble && (isDoubleBubbleCapsule(props.source) || isDoubleBubbleCapsule(props.target))) {
    const topicId = props.source === 'left-topic' || props.source === 'right-topic' ? props.source : props.target === 'left-topic' || props.target === 'right-topic' ? props.target : null
    const capsuleId = isDoubleBubbleCapsule(props.source) ? props.source : isDoubleBubbleCapsule(props.target) ? props.target : null
    const topicNode = props.source === topicId ? sourceNode : targetNode
    const capsuleNode = props.source === capsuleId ? sourceNode : targetNode

    const topicW = topicNode.dimensions?.width ?? topicNode.width ?? topicNode.data?.style?.size ?? 80
    const topicH = topicNode.dimensions?.height ?? topicNode.height ?? topicNode.data?.style?.size ?? 80
    const topicCenterX = topicNode.position.x + Number(topicW) / 2
    const topicCenterY = topicNode.position.y + Number(topicH) / 2
    const capW = Number(capsuleNode.dimensions?.width ?? capsuleNode.width ?? capsuleNode.data?.style?.width ?? 80)
    const capH = Number(capsuleNode.dimensions?.height ?? capsuleNode.height ?? capsuleNode.data?.style?.height ?? 40)
    const capLeft = capsuleNode.position.x
    const capTop = capsuleNode.position.y
    const capCenterY = capTop + capH / 2
    const capsuleLeftEnd = { x: capLeft, y: capCenterY }
    const capsuleRightEnd = { x: capLeft + capW, y: capCenterY }

    const topicRadius = Math.min(Number(topicW), Number(topicH)) / 2
    const isTopicSource = props.source === topicId
    const isMiddle = capsuleId != null && isSimilarityNode(capsuleId)

    if (topicId === 'left-topic') {
      const useLeft = isMiddle
      endX = useLeft ? capsuleLeftEnd.x : capsuleRightEnd.x
      endY = useLeft ? capsuleLeftEnd.y : capsuleRightEnd.y
      const dx = endX - topicCenterX
      const dy = endY - topicCenterY
      const dist = Math.sqrt(dx * dx + dy * dy) || 1
      startX = topicCenterX + (dx / dist) * topicRadius
      startY = topicCenterY + (dy / dist) * topicRadius
    } else if (topicId === 'right-topic') {
      const useRight = isMiddle
      endX = useRight ? capsuleRightEnd.x : capsuleLeftEnd.x
      endY = useRight ? capsuleRightEnd.y : capsuleLeftEnd.y
      const dx = endX - topicCenterX
      const dy = endY - topicCenterY
      const dist = Math.sqrt(dx * dx + dy * dy) || 1
      startX = topicCenterX + (dx / dist) * topicRadius
      startY = topicCenterY + (dy / dist) * topicRadius
    } else {
      // Fallback: not topic–capsule, use standard radial
      const dx = targetCenterX - sourceCenterX
      const dy = targetCenterY - sourceCenterY
      const distance = Math.sqrt(dx * dx + dy * dy) || 1
      const nx = dx / distance
      const ny = dy / distance
      const sourceRadius = Math.min(Number(sourceWidth), Number(sourceHeight)) / 2
      const targetRadius = Math.min(Number(targetWidth), Number(targetHeight)) / 2
      startX = sourceCenterX + nx * sourceRadius
      startY = sourceCenterY + ny * sourceRadius
      endX = targetCenterX - nx * targetRadius
      endY = targetCenterY - ny * targetRadius
    }

    if (topicId != null && !isTopicSource) {
      ;[startX, startY, endX, endY] = [endX, endY, startX, startY]
    }
  } else {
    const dx = targetCenterX - sourceCenterX
    const dy = targetCenterY - sourceCenterY
    const distance = Math.sqrt(dx * dx + dy * dy)

    if (distance === 0) {
      return { edgePath: '', labelX: sourceCenterX, labelY: sourceCenterY }
    }

    const nx = dx / distance
    const ny = dy / distance
    const sourceRadius = Math.min(Number(sourceWidth), Number(sourceHeight)) / 2
    const targetRadius = Math.min(Number(targetWidth), Number(targetHeight)) / 2

    startX = sourceCenterX + nx * sourceRadius
    startY = sourceCenterY + ny * sourceRadius
    endX = targetCenterX - nx * targetRadius
    endY = targetCenterY - ny * targetRadius
  }

  const edgePath = `M ${startX} ${startY} L ${endX} ${endY}`
  const labelX = (startX + endX) / 2
  const labelY = (startY + endY) / 2

  return { edgePath, labelX, labelY }
})

const edgeStyle = computed(() => ({
  stroke: props.data?.style?.strokeColor || '#888888',
  strokeWidth: props.data?.style?.strokeWidth || 2,
}))
</script>

<template>
  <path
    :id="id"
    class="vue-flow__edge-path radial-edge"
    :d="path.edgePath"
    :style="edgeStyle"
  />
</template>

<style scoped>
.radial-edge {
  fill: none;
  transition: stroke 0.2s ease;
}

.radial-edge:hover {
  stroke: #666666;
}
</style>
