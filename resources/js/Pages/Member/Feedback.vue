<script setup>
import { ref, computed } from 'vue'
import { Head, router } from '@inertiajs/vue3'
import MemberLayout from '@/Layouts/MemberLayout.vue'
import FeedbackModal from '@/Pages/Member/FeedbackModal.vue'
import { useAuthModal } from '@/Composables/useAuthModal'

const props = defineProps({
    boards: Array,
    statuses: Array,
    feedbacks: Array,
    upvotedIds: Array,
})

const { requireAuth } = useAuthModal()

const showFeedbackModal = ref(false)

// Search and filter state
const searchQuery = ref('')
const selectedBoard = ref(null)
const selectedStatuses = ref([])
const sortBy = ref('latest') // 'latest', 'votes'
const showStatusDropdown = ref(false)

const toggleStatus = (statusId) => {
    const index = selectedStatuses.value.indexOf(statusId)
    if (index === -1) {
        selectedStatuses.value.push(statusId)
    } else {
        selectedStatuses.value.splice(index, 1)
    }
}

const isStatusSelected = (statusId) => {
    return selectedStatuses.value.includes(statusId)
}

const filteredFeedbacks = computed(() => {
    let result = props.feedbacks

    // Filter by board
    if (selectedBoard.value) {
        result = result.filter(f => f.board?.id === selectedBoard.value)
    }

    // Filter by search query
    if (searchQuery.value.trim()) {
        const query = searchQuery.value.toLowerCase().trim()
        result = result.filter(f =>
            f.title?.toLowerCase().includes(query) ||
            f.description?.toLowerCase().includes(query)
        )
    }

    // Filter by selected statuses
    if (selectedStatuses.value.length > 0) {
        result = result.filter(f => selectedStatuses.value.includes(f.status?.id))
    }

    // Sort
    result = [...result].sort((a, b) => {
        switch (sortBy.value) {
            case 'latest':
                return new Date(b.created_at) - new Date(a.created_at)
            case 'votes':
                return (b.upvotes_count || 0) - (a.upvotes_count || 0)
            default:
                return 0
        }
    })

    return result
})

// Track upvoted items locally (initialized from server)
const upvotedItems = ref(new Set(props.upvotedIds || []))

// Track upvote counts locally for immediate UI updates
const upvoteCounts = ref(
    Object.fromEntries(props.feedbacks.map(f => [f.id, f.upvotes_count || 0]))
)

// Track which items are currently being upvoted
const upvotingItems = ref(new Set())

const toggleVote = async (feedbackId) => {
    if (!requireAuth()) return
    if (upvotingItems.value.has(feedbackId)) return
    upvotingItems.value.add(feedbackId)

    try {
        const response = await fetch(`/feedback/${feedbackId}/upvote`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]')?.content,
                'Accept': 'application/json',
            },
        })

        const data = await response.json()

        if (data.success) {
            if (data.upvoted) {
                upvotedItems.value.add(feedbackId)
            } else {
                upvotedItems.value.delete(feedbackId)
            }
            upvoteCounts.value[feedbackId] = data.upvoteCount
        }
    } catch (error) {
        console.error('Failed to toggle upvote:', error)
    } finally {
        upvotingItems.value.delete(feedbackId)
    }
}

const isUpvoted = (feedbackId) => {
    return upvotedItems.value.has(feedbackId)
}

const getUpvoteCount = (feedbackId) => {
    return upvoteCounts.value[feedbackId] ?? 0
}

const shortenDescription = (text, maxLength = 110) => {
    if (!text) return ''
    if (text.length <= maxLength) return text
    return text.substring(0, maxLength) + '...'
}

const selectBoard = (boardId) => {
    selectedBoard.value = selectedBoard.value === boardId ? null : boardId
}

const openFeedbackModal = () => {
    if (!requireAuth()) return
    showFeedbackModal.value = true
}

const currentBoardName = computed(() =>
    selectedBoard.value
        ? props.boards.find(b => b.id === selectedBoard.value)?.name
        : 'All Requests'
)
</script>

<template>
    <Head title="Feedback" />

    <MemberLayout>
        <div class="mx-auto flex max-w-7xl gap-12 px-8 py-8">
            <!-- Left Sidebar -->
            <aside class="w-64 flex-shrink-0 self-start">
                <div class="space-y-7">
                    <!-- Submit Button -->
                    <button
                        @click="openFeedbackModal"
                        class="mt-2 flex w-full items-center justify-center gap-2 rounded-xl bg-[#b2a23c] px-4 py-3 text-sm font-medium text-white transition-transform duration-200 hover:-translate-y-0.5 hover:bg-[#a29335]"
                    >
                        <svg class="h-5 w-5" stroke-width="1.5">
                            <use href="/images/icons.svg#plus" />
                        </svg>
                        Submit Idea
                    </button>

                    <!-- Boards Section -->
                    <div>
                        <h3 class="mb-4 text-xs font-semibold uppercase tracking-[0.18em] text-slate-400">Boards</h3>
                        <div class="space-y-2">
                        <button
                            @click="selectedBoard = null"
                            :class="[
                                'group flex w-full items-center gap-3 rounded-xl border px-4 py-3 text-sm font-medium transition-colors',
                                !selectedBoard ? 'border-slate-200 bg-white text-slate-900 shadow-sm' : 'border-transparent text-slate-700 hover:bg-slate-50'
                            ]"
                        >
                            <span
                                class="h-2.5 w-2.5 rounded-full bg-black ring-4 ring-black/10 transition-all group-hover:ring-black/20"
                            ></span>
                            <span class="flex-1 text-left">All Requests</span>
                            <span class="text-xs text-slate-500">{{ feedbacks.length }}</span>
                        </button>
                        <button
                            v-for="board in boards"
                            :key="board.id"
                            @click="selectBoard(board.id)"
                            :class="[
                                'group flex w-full items-center gap-3 rounded-xl border px-4 py-3 text-sm font-medium transition-colors',
                                selectedBoard === board.id ? 'border-slate-200 bg-white text-slate-900 shadow-sm' : 'border-transparent text-slate-700 hover:bg-slate-50'
                            ]"
                        >
                            <span
                                class="h-2.5 w-2.5 rounded-full ring-4 ring-opacity-10 transition-all group-hover:ring-opacity-20"
                                :class="$boardColorClasses[board.color] || 'bg-gray-500 ring-gray-500'"
                            ></span>
                            <span class="flex-1 text-left">{{ board.name }}</span>
                            <span class="text-xs text-slate-500">{{ board.feedback_count }}</span>
                        </button>
                    </div>
                </div>
                </div>
            </aside>

            <!-- Main Content -->
            <main class="flex-1">
                <!-- Header -->
                <div class="mb-7">
                    <div class="mb-2 flex items-start justify-between gap-6">
                        <div>
                            <h1 class="text-5xl font-semibold tracking-tight text-slate-900">
                                {{ currentBoardName }}
                            </h1>
                            <p class="mt-3 text-lg text-slate-500">Vote on existing requests or suggest a new feature.</p>
                        </div>
                        <div class="flex items-center gap-3">
                            <div class="relative min-w-[230px]">
                                <input
                                    v-model="searchQuery"
                                    type="text"
                                    placeholder="Search posts..."
                                    class="h-11 w-full rounded-xl border border-slate-200 bg-white pl-10 pr-4 text-sm text-slate-700 shadow-sm outline-none transition focus:border-slate-300 focus:ring-2 focus:ring-slate-200">
                                <svg class="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-slate-400" stroke-width="2">
                                    <use href="/images/icons.svg#search" />
                                </svg>
                            </div>

                            <!-- Status Filter -->
                            <div class="relative">
                                <button
                                    @click="showStatusDropdown = !showStatusDropdown"
                                    class="flex h-11 w-11 items-center justify-center rounded-xl border border-slate-200 bg-white shadow-sm transition-colors hover:bg-slate-50"
                                >
                                    <svg class="h-5 w-5 text-slate-600" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                                        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M3 4a1 1 0 011-1h16a1 1 0 011 1v2.586a1 1 0 01-.293.707l-6.414 6.414a1 1 0 00-.293.707V17l-4 4v-6.586a1 1 0 00-.293-.707L3.293 7.293A1 1 0 013 6.586V4z"/>
                                    </svg>
                                </button>

                                <!-- Dropdown Menu -->
                                <div
                                    v-if="showStatusDropdown"
                                    class="absolute right-0 top-full z-30 mt-2 w-56 rounded-xl border border-slate-200 bg-white shadow-lg"
                                >
                                    <div class="p-2">
                                        <button
                                            v-for="status in statuses"
                                            :key="status.id"
                                            @click="toggleStatus(status.id)"
                                            :class="[
                                                'flex w-full items-center gap-2 rounded-lg px-3 py-2 text-sm transition-colors',
                                                isStatusSelected(status.id) ? 'bg-slate-100' : 'hover:bg-slate-50'
                                            ]"
                                        >
                                            <span
                                                :class="[
                                                    'flex h-4 w-4 items-center justify-center rounded border-2',
                                                    isStatusSelected(status.id) ? 'border-slate-900 bg-slate-900' : 'border-slate-300'
                                                ]"
                                            >
                                                <svg v-if="isStatusSelected(status.id)" class="w-2.5 h-2.5 text-white" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                                                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="3" d="M5 13l4 4L19 7"/>
                                                </svg>
                                            </span>
                                            <span class="w-2 h-2 rounded-full" :class="$statusDotColors[status.color]"></span>
                                            <span class="text-slate-700">{{ status.name }}</span>
                                        </button>
                                    </div>
                                    <div v-if="selectedStatuses.length > 0" class="border-t border-slate-100 p-2">
                                        <button
                                            @click="selectedStatuses = []; showStatusDropdown = false"
                                            class="w-full py-1 text-center text-sm text-slate-500 hover:text-slate-700"
                                        >
                                            Clear all
                                        </button>
                                    </div>
                                </div>

                                <!-- Click outside to close -->
                                <div v-if="showStatusDropdown" class="fixed inset-0 z-20" @click="showStatusDropdown = false"></div>
                            </div>

                            <!-- Sort Buttons -->
                            <div class="flex items-center gap-1 rounded-xl border border-slate-200 bg-white p-1 shadow-sm">
                                <button
                                    @click="sortBy = 'latest'"
                                    :class="[
                                        'rounded-lg px-4 py-2 text-sm font-medium transition-colors',
                                        sortBy === 'latest' ? 'bg-[#b2a23c] text-white' : 'text-slate-500 hover:bg-[#f4efcf] hover:text-[#6f6722]'
                                    ]"
                                >
                                    Latest
                                </button>
                                <button
                                    @click="sortBy = 'votes'"
                                    :class="[
                                        'rounded-lg px-4 py-2 text-sm font-medium transition-colors',
                                        sortBy === 'votes' ? 'bg-[#b2a23c] text-white' : 'text-slate-500 hover:bg-[#f4efcf] hover:text-[#6f6722]'
                                    ]"
                                >
                                    Upvoted
                                </button>
                            </div>
                        </div>
                    </div>
                </div>

                <!-- Feedback List -->
                <div class="space-y-4">
                    <div
                        v-for="feedback in filteredFeedbacks"
                        :key="feedback.id"
                        @click="router.visit(`/feedback/${feedback.slug}`)"
                        class="cursor-pointer rounded-2xl border border-slate-200 bg-white px-4 py-4 shadow-sm transition-all hover:border-slate-300 hover:shadow-md"
                    >
                        <div class="flex gap-4">
                            <div class="flex w-12 flex-shrink-0 flex-col items-center gap-1 pt-1">
                                <button
                                    @click.stop="toggleVote(feedback.id)"
                                    :class="[
                                        'flex h-10 w-10 items-center justify-center rounded-xl border transition-all',
                                        isUpvoted(feedback.id) ? 'border-[#b2a23c] bg-[#b2a23c]' : 'border-slate-200 bg-white hover:border-[#cbbb5a]'
                                    ]"
                                >
                                    <svg class="h-5 w-5 transition-colors" :class="isUpvoted(feedback.id) ? 'text-white' : 'text-slate-600'" stroke-width="1.5">
                                        <use href="/images/icons.svg#upvote" />
                                    </svg>
                                </button>
                                <span class="text-xl font-semibold leading-none text-slate-700">{{ getUpvoteCount(feedback.id) }}</span>
                            </div>
                            <div class="min-w-0 flex-1">
                                <div class="flex items-start justify-between gap-4">
                                    <div class="min-w-0">
                                        <h3 class="mb-1 text-2xl font-semibold tracking-tight text-slate-900">{{ feedback.title }}</h3>
                                        <p class="mb-4 text-base text-slate-600">{{ shortenDescription(feedback.description, 170) }}</p>
                                    </div>
                                <span
                                    v-if="feedback.status"
                                    :class="['inline-flex flex-shrink-0 items-center gap-1.5 rounded-xl px-3 py-1.5 text-xs font-medium border-[0.25px]', $statusClasses[feedback.status.color]]"
                                >
                                    <span class="w-1.5 h-1.5 bg-current rounded-full"></span>
                                    {{ feedback.status.name }}
                                </span>
                                </div>
                                <div class="flex items-center gap-5 text-sm text-slate-500">
                                    <div class="flex items-center gap-1.5">
                                        <svg class="h-4 w-4" stroke-width="1.5">
                                            <use href="/images/icons.svg#comment" />
                                        </svg>
                                        <span>{{ feedback.comments_count }}</span>
                                    </div>
                                    <div class="flex items-center gap-2">
                                        <img
                                            :src="feedback.author?.avatar || 'https://xsgames.co/randomusers/assets/avatars/male/23.jpg'"
                                            :alt="feedback.author?.name"
                                            class="h-6 w-6 rounded-full border border-slate-200"
                                        >
                                        <span>{{ feedback.author?.name }}</span>
                                    </div>
                                </div>
                            </div>
                        </div>
                    </div>

                    <!-- Empty State -->
                    <div v-if="filteredFeedbacks.length === 0" class="py-16 text-center">
                        <svg class="w-16 h-16 text-gray-300 mx-auto mb-4" fill="none" viewBox="0 0 24 24" stroke="currentColor">
                            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M20 13V6a2 2 0 00-2-2H6a2 2 0 00-2 2v7m16 0v5a2 2 0 01-2 2H6a2 2 0 01-2-2v-5m16 0h-2.586a1 1 0 00-.707.293l-2.414 2.414a1 1 0 01-.707.293h-3.172a1 1 0 01-.707-.293l-2.414-2.414A1 1 0 006.586 13H4"/>
                        </svg>
                        <h3 class="text-lg font-medium text-gray-900 mb-1">No feedback found</h3>
                        <p class="text-sm text-gray-500">Try adjusting your search or filters</p>
                    </div>
                </div>
            </main>
        </div>

        <!-- Submit Feedback Modal -->
        <FeedbackModal
            v-model:show="showFeedbackModal"
            :boards="boards"
            :statuses="statuses"
            :feedbacks="feedbacks"
        />
    </MemberLayout>
</template>
