<script lang="ts">
  import { slide } from 'svelte/transition';
  import type { Post, PostType, Comment, Reply, Media } from '$lib/types/post';
  import type { SurfaceColor } from '$lib/types/theme';
  
  export let type: PostType = 'default';
  
  $: cardClasses = `
    bg-surface-50 dark:bg-surface-dark-500 backdrop-blur-sm rounded-xl 
    border border-surface-200 dark:border-surface-dark-400 
    overflow-hidden transition-all duration-300 
    hover:shadow-md 
    ${type === 'featured' ? 'shadow-lg' : 'shadow-sm'}
    ${type === 'compact' ? 'p-4' : 'p-6'}
  `;
  
  export let title: Post['title'];
  export let author: Post['author'];
  export let timeAgo: Post['timeAgo'];
  export let content: Post['content'];
  export let upvotes: Post['upvotes'];
  export let commentCount: Post['commentCount'];
  export let media: Media | undefined = undefined;

  let isExpanded = false;
  let isUpvoted = false;
  let upvoteCount: number = upvotes;
  let newComment = '';
  let replyingTo: number | null = null;
  let replyText = '';
  let isSubmitting = false;

  // Truncate content for preview
  $: previewContent = content.slice(0, 180) + (content.length > 180 ? '...' : '');

  type SortOption = 'newest' | 'oldest' | 'most-enlightened' | 'most-replies';
  type FilterOption = 'all' | 'verified' | 'enlightened';

  let sortBy: SortOption = 'newest';
  let filterBy: FilterOption = 'all';

  // Example comments with more variety
  let mockComments: Comment[] = [
    {
      id: 1,
      author: "Alex Thompson",
      isVerified: true,
      content: "This is particularly interesting when you consider the implications for distributed systems.",
      upvotes: 42,
      isEnlightened: true,
      timeAgo: "1h ago",
      replies: [
        {
          id: 101,
          author: "Sarah Chen",
          isVerified: true,
          content: "Excellent point about distributed systems!",
          timeAgo: "45m ago",
          upvotes: 12,
          isEnlightened: true
        }
      ]
    },
    {
      id: 2,
      author: "Maria Garcia",
      isVerified: true,
      content: "Great analysis. I'd add that the performance implications are significant.",
      upvotes: 28,
      isEnlightened: false,
      timeAgo: "2h ago",
      replies: []
    },
    {
      id: 3,
      author: "John Doe",
      isVerified: false,
      content: "Could you elaborate more on the technical aspects?",
      upvotes: 15,
      isEnlightened: true,
      timeAgo: "3h ago",
      replies: []
    }
  ];

  // Add pagination state
  let commentsPerPage = 5;
  let currentPage = 1;
  let isLoadingMore = false;
  let hasMoreComments = true;

  // Computed value for filtered and sorted comments
  $: filteredComments = mockComments.filter(comment => {
    switch (filterBy) {
      case 'verified':
        return comment.isVerified;
      case 'enlightened':
        return comment.isEnlightened;
      default:
        return true;
    }
  });

  $: sortedComments = [...filteredComments].sort((a, b) => {
    switch (sortBy) {
      case 'oldest':
        return a.id - b.id;
      case 'most-enlightened':
        return b.upvotes - a.upvotes;
      case 'most-replies':
        return b.replies.length - a.replies.length;
      default: // newest
        return b.id - a.id;
    }
  });

  // Update visible comments to use sorted and filtered list
  $: visibleComments = sortedComments.slice(0, currentPage * commentsPerPage);
  $: remainingComments = sortedComments.length - visibleComments.length;

  function handleUpvote(): void {
    isUpvoted = !isUpvoted;
    upvoteCount += isUpvoted ? 1 : -1;
  }

  function getProviderIcon(provider: Media['provider']): string | null {
    switch (provider?.toLowerCase()) {
      case 'youtube':
        return 'M21.582,6.186c-0.23-0.86-0.908-1.538-1.768-1.768C18.254,4,12,4,12,4S5.746,4,4.186,4.418 c-0.86,0.23-1.538,0.908-1.768,1.768C2,7.746,2,12,2,12s0,4.254,0.418,5.814c0.23,0.86,0.908,1.538,1.768,1.768 C5.746,20,12,20,12,20s6.254,0,7.814-0.418c0.861-0.23,1.538-0.908,1.768-1.768C22,16.254,22,12,22,12S22,7.746,21.582,6.186z M10,15.464V8.536L16,12L10,15.464z';
      case 'twitter':
        return 'M23.643 4.937c-.835.37-1.732.62-2.675.733.962-.576 1.7-1.49 2.048-2.578-.9.534-1.897.922-2.958 1.13-.85-.904-2.06-1.47-3.4-1.47-2.572 0-4.658 2.086-4.658 4.66 0 .364.042.718.12 1.06-3.873-.195-7.304-2.05-9.602-4.868-.4.69-.63 1.49-.63 2.342 0 1.616.823 3.043 2.072 3.878-.764-.025-1.482-.234-2.11-.583v.06c0 2.257 1.605 4.14 3.737 4.568-.392.106-.803.162-1.227.162-.3 0-.593-.028-.877-.082.593 1.85 2.313 3.198 4.352 3.234-1.595 1.25-3.604 1.995-5.786 1.995-.376 0-.747-.022-1.112-.065 2.062 1.323 4.51 2.093 7.14 2.093 8.57 0 13.255-7.098 13.255-13.254 0-.2-.005-.402-.014-.602.91-.658 1.7-1.477 2.323-2.41z';
      default:
        return null;
    }
  }

  async function handleSubmitComment(): Promise<void> {
    if (!newComment.trim()) return;
    
    isSubmitting = true;
    try {
      mockComments = [
        {
          id: Date.now(),
          author: "You",
          content: newComment,
          upvotes: 1,
          timeAgo: "Just now",
          replies: []
        },
        ...mockComments
      ];
      newComment = '';
    } finally {
      isSubmitting = false;
    }
  }

  async function handleSubmitReply(commentId: number): Promise<void> {
    if (!replyText.trim()) return;
    
    isSubmitting = true;
    try {
      mockComments = mockComments.map(comment => {
        if (comment.id === commentId) {
          return {
            ...comment,
            replies: [
              {
                id: Date.now(),
                author: "You",
                content: replyText,
                timeAgo: "Just now",
                upvotes: 1
              },
              ...comment.replies
            ]
          };
        }
        return comment;
      });
      replyText = '';
      replyingTo = null;
    } finally {
      isSubmitting = false;
    }
  }

  function toggleCommentEnlighten(commentId: number): void {
    mockComments = mockComments.map(comment => {
      if (comment.id === commentId) {
        return {
          ...comment,
          isEnlightened: !comment.isEnlightened,
          upvotes: comment.upvotes + (comment.isEnlightened ? -1 : 1)
        };
      }
      return comment;
    });
  }

  // Load more comments function
  async function loadMoreComments() {
    if (isLoadingMore || !hasMoreComments) return;
    
    isLoadingMore = true;
    try {
      // Simulate API call delay
      await new Promise(resolve => setTimeout(resolve, 500));
      
      currentPage++;
      hasMoreComments = visibleComments.length < sortedComments.length;
    } finally {
      isLoadingMore = false;
    }
  }

  // Intersection Observer for infinite scroll
  let commentListContainer: HTMLElement;
  
  function setupIntersectionObserver() {
    if (typeof IntersectionObserver === 'undefined') return;
    
    const observer = new IntersectionObserver(
      (entries) => {
        const [entry] = entries;
        if (entry.isIntersecting && hasMoreComments && !isLoadingMore) {
          loadMoreComments();
        }
      },
      { threshold: 0.1 }
    );

    if (commentListContainer) {
      observer.observe(commentListContainer);
    }

    return () => observer.disconnect();
  }
</script>

<style>
  .glow {
    filter: drop-shadow(0 0 8px rgb(234, 179, 8));
    transition: filter 0.3s ease;
  }
</style>

<article class={cardClasses}>
  <div class="p-6">
    <div class="flex items-start gap-6">
      <button 
        class="flex flex-col items-center gap-1.5 p-2.5 w-[3.75rem] rounded-lg 
          hover:bg-surface-100 dark:hover:bg-surface-dark-300 transition-all"
        on:click={handleUpvote}
      >
        <svg 
          xmlns="http://www.w3.org/2000/svg" 
          class="h-5 w-5 transition-all duration-300 {isUpvoted ? 'text-yellow-500 glow' : 'text-surface-400'}" 
          fill={isUpvoted ? 'currentColor' : 'none'} 
          viewBox="0 0 24 24" 
          stroke="currentColor"
        >
          <path 
            stroke-linecap="round" 
            stroke-linejoin="round" 
            stroke-width="2" 
            d="M13 10V3L4 14h7v7l9-11h-7z"
          />
        </svg>
        <span class="text-sm font-medium transition-colors duration-300 
          {isUpvoted ? 'text-yellow-500' : 'text-surface-600 dark:text-surface-dark-200'}">
          {upvoteCount}
        </span>
      </button>

      <div class="flex-1 min-w-0 space-y-4">
        <div class="space-y-2.5">
          <div class="flex items-center gap-2 text-sm text-surface-500 dark:text-surface-dark-100">
            <a href="/profile/{author}" class="font-medium text-surface-900 dark:text-white hover:text-primary-500 transition-colors">
              {author}
            </a>
            <span>•</span>
            <span class="text-surface-600 dark:text-surface-dark-100">{timeAgo}</span>
          </div>
          <h2 class="text-xl font-semibold text-surface-900 dark:text-surface-dark-50">
            <a href="/post/{title.toLowerCase().replace(/\s+/g, '-')}" class="hover:text-primary-500 transition-colors">
              {title}
            </a>
          </h2>
        </div>
        
        {#if media}
          <div class="relative aspect-video rounded-lg overflow-hidden bg-surface-100 dark:bg-surface-dark-300">
            {#if media.type === 'image'}
              <img src={media.url} alt="" class="w-full h-full object-cover">
            {:else if media.type === 'gallery' && Array.isArray(media.url)}
              <div class="grid grid-cols-2 gap-1 h-full">
                <img src={media.url[0]} alt="" class="w-full h-full object-cover">
                <div class="relative">
                  <img src={media.url[1]} alt="" class="w-full h-full object-cover">
                  {#if media.url.length > 2}
                    <div class="absolute inset-0 bg-black/50 flex items-center justify-center text-white font-medium">
                      +{media.url.length - 2}
                    </div>
                  {/if}
                </div>
              </div>
            {:else if media.type === 'video'}
              <div class="relative w-full h-full">
                <!-- Add video player implementation -->
              </div>
            {/if}
          </div>
        {/if}

        <p class="text-surface-600 dark:text-surface-dark-50 leading-relaxed">
          {isExpanded ? content : previewContent}
        </p>

        {#if isExpanded}
          <div transition:slide={{ duration: 300 }} class="space-y-6">
            <div class="h-px bg-surface-200 dark:bg-surface-dark-300"></div>
            
            <div class="space-y-6">
              <div class="flex items-center justify-between mb-4">
                <h3 class="text-lg font-medium text-surface-900 dark:text-surface-dark-50">
                  Comments ({sortedComments.length})
                </h3>
                
                <div class="flex items-center gap-3">
                  <!-- Filter dropdown -->
                  <select
                    bind:value={filterBy}
                    class="px-3 py-1.5 text-sm rounded-lg bg-surface-100 dark:bg-surface-dark-300 
                      border border-surface-200 dark:border-surface-dark-400
                      text-surface-900 dark:text-surface-dark-50"
                  >
                    <option value="all">All Comments</option>
                    <option value="verified">Verified Only</option>
                    <option value="enlightened">Enlightened Only</option>
                  </select>

                  <!-- Sort dropdown -->
                  <select
                    bind:value={sortBy}
                    class="px-3 py-1.5 text-sm rounded-lg bg-surface-100 dark:bg-surface-dark-300 
                      border border-surface-200 dark:border-surface-dark-400
                      text-surface-900 dark:text-surface-dark-50"
                  >
                    <option value="newest">Newest First</option>
                    <option value="oldest">Oldest First</option>
                    <option value="most-enlightened">Most Enlightened</option>
                    <option value="most-replies">Most Replies</option>
                  </select>
                </div>
              </div>

              <!-- Comment list with infinite scroll -->
              <div 
                class="space-y-6"
                bind:this={commentListContainer}
                use:setupIntersectionObserver
              >
                {#if visibleComments.length === 0}
                  <div class="text-center py-8 text-surface-500 dark:text-surface-dark-200">
                    No comments match your filters
                  </div>
                {:else}
                  {#each visibleComments as comment (comment.id)}
                    <div class="space-y-4 pl-4 border-l-2 border-surface-200 dark:border-surface-dark-300">
                      <div class="space-y-2">
                        <div class="flex items-center gap-2 text-sm">
                          <span class="font-medium text-surface-900 dark:text-white">
                            {comment.author}
                            {#if comment.isVerified}
                              <span class="inline-flex ml-1" title="Verified">
                                <svg class="w-4 h-4 text-primary-500" viewBox="0 0 20 20" fill="currentColor">
                                  <path fill-rule="evenodd" d="M10 18a8 8 0 100-16 8 8 0 000 16zm3.707-9.293a1 1 0 00-1.414-1.414L9 10.586 7.707 9.293a1 1 0 00-1.414 1.414l2 2a1 1 0 001.414 0l4-4z" clip-rule="evenodd" />
                                </svg>
                              </span>
                            {/if}
                          </span>
                          <span class="text-surface-500 dark:text-surface-dark-50">{comment.timeAgo}</span>
                        </div>
                        <p class="text-surface-600 dark:text-surface-dark-50">{comment.content}</p>
                        <div class="flex items-center gap-4 text-sm text-surface-500">
                          <button 
                            class="flex items-center gap-1.5 hover:text-yellow-500 transition-colors"
                            on:click={() => toggleCommentEnlighten(comment.id)}
                          >
                            <svg 
                              xmlns="http://www.w3.org/2000/svg" 
                              class="h-4 w-4 transition-all duration-300 {comment.isEnlightened ? 'text-yellow-500 glow' : ''}" 
                              fill={comment.isEnlightened ? 'currentColor' : 'none'} 
                              viewBox="0 0 24 24" 
                              stroke="currentColor"
                            >
                              <path 
                                stroke-linecap="round" 
                                stroke-linejoin="round" 
                                stroke-width="2" 
                                d="M13 10V3L4 14h7v7l9-11h-7z"
                              />
                            </svg>
                            <span class="transition-colors duration-300 {comment.isEnlightened ? 'text-yellow-500' : ''}">
                              {comment.upvotes}
                            </span>
                          </button>
                          <button 
                            class="hover:text-primary-500 transition-colors"
                            on:click={() => replyingTo = replyingTo === comment.id ? null : comment.id}
                          >
                            Reply
                          </button>
                        </div>
                      </div>

                      {#if replyingTo === comment.id}
                        <div class="flex gap-3 pl-4" transition:slide={{ duration: 200 }}>
                          <div class="flex-1">
                            <textarea
                              bind:value={replyText}
                              placeholder="Write a reply..."
                              class="w-full px-3 py-2 text-sm 
                                bg-surface-100 dark:bg-surface-dark-300 
                                rounded-lg border border-surface-200 dark:border-surface-dark-400
                                focus:outline-none focus:ring-2 focus:ring-primary-500/50
                                placeholder-surface-500 dark:placeholder-surface-dark-300
                                text-surface-900 dark:text-surface-dark-50
                                resize-none"
                              rows="2"
                            ></textarea>
                          </div>
                          <div class="flex flex-col gap-2">
                            <button 
                              on:click={() => handleSubmitReply(comment.id)}
                              disabled={!replyText.trim() || isSubmitting}
                              class="px-4 py-2 text-sm font-medium rounded-lg
                                bg-primary-600 hover:bg-primary-700 disabled:opacity-50 disabled:cursor-not-allowed
                                text-white transition-colors"
                            >
                              Reply
                            </button>
                            <button 
                              on:click={() => replyingTo = null}
                              class="px-4 py-2 text-sm font-medium rounded-lg
                                bg-surface-200 hover:bg-surface-300 
                                dark:bg-surface-dark-300 dark:hover:bg-surface-dark-400
                                text-surface-700 dark:text-surface-dark-50 
                                transition-colors"
                            >
                              Cancel
                            </button>
                          </div>
                        </div>
                      {/if}

                      {#if comment.replies.length > 0}
                        <div class="pl-4 space-y-3">
                          {#each comment.replies as reply}
                            <div class="space-y-2">
                              <div class="flex items-center gap-2 text-sm">
                                <span class="font-medium text-surface-900 dark:text-white">{reply.author}</span>
                                <span class="text-surface-500 dark:text-surface-dark-50">{reply.timeAgo}</span>
                              </div>
                              <p class="text-surface-600 dark:text-surface-dark-50">{reply.content}</p>
                            </div>
                          {/each}
                        </div>
                      {/if}
                    </div>
                  {/each}
                {/if}

                {#if isLoadingMore}
                  <div class="flex justify-center py-4">
                    <div class="animate-pulse text-surface-500 dark:text-surface-dark-200">
                      Loading more comments...
                    </div>
                  </div>
                {/if}

                {#if hasMoreComments && !isLoadingMore}
                  <button
                    class="w-full py-3 text-sm text-primary-500 hover:text-primary-600 transition-colors"
                    on:click={loadMoreComments}
                  >
                    Show {Math.min(remainingComments, commentsPerPage)} more comments
                  </button>
                {/if}
              </div>
            </div>
          </div>
        {/if}

        <div class="flex items-center gap-6 pt-2.5 text-sm text-surface-500 dark:text-surface-dark-50">
          <button class="flex items-center gap-2 hover:text-primary-500 transition-colors">
            <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 12h.01M12 12h.01M16 12h.01M21 12c0 4.418-4.03 8-9 8a9.863 9.863 0 01-4.255-.949L3 20l1.395-3.72C3.512 15.042 3 13.574 3 12c0-4.418 4.03-8 9-8s9 3.582 9 8z" />
            </svg>
            {commentCount} comments
          </button>
          
          <button 
            class="flex items-center gap-2 hover:text-primary-500 transition-colors"
            on:click={() => isExpanded = !isExpanded}
          >
            <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 9l-7 7-7-7" />
            </svg>
            {isExpanded ? 'Show less' : 'Show more'}
          </button>
        </div>
      </div>
    </div>
  </div>
</article> 