<script>
  import { onMount } from 'svelte';
  import * as cheerio from 'cheerio';
  import Prism from 'prismjs';
  import 'prismjs/components/prism-markup.js';

  const STORAGE_KEY_HTML = 'cheerio-playground-html';
  const STORAGE_KEY_CODE = 'cheerio-playground-code';
  const defaultHtml = `<h1>Hello, Cheerio!</h1>
<p class="main">This is a paragraph to parse.</p>
<ul>
    <li class="item">Item 1</li>
    <li class="item">Item 2</li>
    <li class="item">Item 3</li>
</ul>`;

  let htmlInput = $state(defaultHtml);

  let cheerioCode = $state(`$('.item')`);
  let outputItems = $state([]);
  let isError = $state(false);

  let htmlHighlightRef = $state(null);
  let codeHighlightRef = $state(null);

  // Save HTML input to localStorage when it changes (debounced)
  $effect(() => {
    const value = htmlInput;
    const timeout = setTimeout(() => {
      if (typeof localStorage !== 'undefined') {
        localStorage.setItem(STORAGE_KEY_HTML, value);
      }
    }, 1000);
    return () => clearTimeout(timeout);
  });

  // Save Cheerio code to localStorage when it changes (debounced)
  $effect(() => {
    const value = cheerioCode;
    const timeout = setTimeout(() => {
      if (typeof localStorage !== 'undefined') {
        localStorage.setItem(STORAGE_KEY_CODE, value);
      }
    }, 1000);
    return () => clearTimeout(timeout);
  });

  function highlightHtml(code) {
    return Prism.highlight(code, Prism.languages.markup, 'markup');
  }

  let highlightedInput = $state('');
  let highlightedCode = $state('');

  // Debounced syntax highlighting for HTML input
  $effect(() => {
    const value = htmlInput + '\n';
    const timeout = setTimeout(() => {
      highlightedInput = highlightHtml(value);
    }, 50);
    return () => clearTimeout(timeout);
  });

  // Debounced syntax highlighting for code input
  $effect(() => {
    const value = cheerioCode + '\n';
    const timeout = setTimeout(() => {
      highlightedCode = highlightHtml(value);
    }, 50);
    return () => clearTimeout(timeout);
  });

  function syncScroll(textarea, highlightEl) {
    if (highlightEl) {
      highlightEl.scrollTop = textarea.scrollTop;
      highlightEl.scrollLeft = textarea.scrollLeft;
    }
  }

  function prettyPrintHtml(html) {
    const tab = '  ';
    let result = '';
    let indent = 0;
    
    // Normalize and split by tags
    const tokens = html.replace(/>\s+</g, '><').split(/(<[^>]+>)/g).filter(Boolean);
    
    for (const token of tokens) {
      if (token.startsWith('</')) {
        // Closing tag - decrease indent first
        indent = Math.max(0, indent - 1);
        result += tab.repeat(indent) + token + '\n';
      } else if (token.startsWith('<') && !token.startsWith('<!')) {
        // Opening tag
        const isSelfClosing = token.endsWith('/>') || /^<(area|base|br|col|embed|hr|img|input|link|meta|param|source|track|wbr)/i.test(token);
        const isVoid = /^<(area|base|br|col|embed|hr|img|input|link|meta|param|source|track|wbr)/i.test(token);
        
        result += tab.repeat(indent) + token + '\n';
        
        if (!isSelfClosing && !isVoid) {
          indent++;
        }
      } else if (token.trim()) {
        // Text content
        result += tab.repeat(indent) + token.trim() + '\n';
      }
    }
    
    return result.trim();
  }

  function execute() {
    try {
      const doc = cheerio.load(htmlInput);
      const userFunc = new Function('$', `return ${cheerioCode}`);
      const result = userFunc(doc);

      if (result && result.cheerio) {
        if (result.length === 0) {
          outputItems = ['(No elements found)'];
        } else {
          outputItems = result
            .map((index, element) => prettyPrintHtml(doc.html(element)))
            .get();
        }
      } else if (typeof result === 'object' && result !== null) {
        outputItems = [JSON.stringify(result, null, 2)];
      } else {
        outputItems = [String(result)];
      }
      isError = false;
    } catch (error) {
      outputItems = [`Error: ${error.message}`];
      isError = true;
    }
  }

  function handleKeydown(event) {
    if ((event.metaKey || event.ctrlKey) && event.key === 'Enter') {
      event.preventDefault();
      execute();
    }
  }

  onMount(() => {
    // Load saved values from localStorage
    const savedHtml = localStorage.getItem(STORAGE_KEY_HTML);
    const savedCode = localStorage.getItem(STORAGE_KEY_CODE);
    if (savedHtml !== null) {
      htmlInput = savedHtml;
    }
    if (savedCode !== null) {
      cheerioCode = savedCode;
    }
    // Initialize syntax highlighting immediately
    highlightedInput = highlightHtml(htmlInput + '\n');
    highlightedCode = highlightHtml(cheerioCode + '\n');
    execute();
  });
</script>

<svelte:head>
  <title>Cheerio Playground</title>
  <meta name="description" content="Test and experiment with Cheerio selectors in real-time" />
</svelte:head>

<div class="playground">
  <header>
    <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
      <path d="M14.5 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V7.5L14.5 2z" />
      <polyline points="14,2 14,8 20,8" />
      <line x1="16" y1="13" x2="8" y2="13" />
      <line x1="16" y1="17" x2="8" y2="17" />
    </svg>
    <h1>Cheerio Playground</h1>
    <span class="separator">·</span>
    <p class="tagline">Test cheerio selectors in real-time</p>
  </header>

  <main>
    <div class="panel html-panel">
      <div class="panel-header">
        <span class="panel-icon">
          <svg viewBox="0 0 24 24" fill="currentColor">
            <path d="M12 17.27L18.18 21l-1.64-7.03L22 9.24l-7.19-.61L12 2 9.19 8.63 2 9.24l5.46 4.73L5.82 21z"/>
          </svg>
        </span>
        <label for="html-input">HTML Input</label>
        <button class="pretty-print-btn" onclick={() => htmlInput = prettyPrintHtml(htmlInput)} title="Pretty print HTML">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <path d="M4 7h16M4 12h16M4 17h10"/>
          </svg>
          <span>Format</span>
        </button>
      </div>
      <div class="editor-container">
        <pre class="editor-highlight" aria-hidden="true" bind:this={htmlHighlightRef}><code>{@html highlightedInput}</code></pre>
        <textarea
          id="html-input"
          bind:value={htmlInput}
          onkeydown={handleKeydown}
          onscroll={(e) => syncScroll(e.target, htmlHighlightRef)}
          placeholder="Enter HTML here..."
          spellcheck="false"
        ></textarea>
      </div>
    </div>

    <div class="panel code-panel">
      <div class="panel-header">
        <span class="panel-icon">
          <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2">
            <polyline points="16,18 22,12 16,6" />
            <polyline points="8,6 2,12 8,18" />
          </svg>
        </span>
        <label for="cheerio-code">Cheerio Code</label>
        <span class="hint">Use $ to access the loaded document</span>
      </div>
      <div class="editor-container">
        <pre class="editor-highlight" aria-hidden="true" bind:this={codeHighlightRef}><code>{@html highlightedCode}</code></pre>
        <textarea
          id="cheerio-code"
          bind:value={cheerioCode}
          onkeydown={handleKeydown}
          onscroll={(e) => syncScroll(e.target, codeHighlightRef)}
          placeholder="$('.selector')"
          spellcheck="false"
        ></textarea>
      </div>
    </div>

    <button class="run-button" onclick={execute}>
      <svg viewBox="0 0 24 24" fill="currentColor">
        <polygon points="5,3 19,12 5,21" />
      </svg>
      <span>Run Cheerio Code</span>
      <kbd>⌘↵</kbd>
    </button>

    <div class="panel output-panel">
      <div class="panel-header">
        <span class="panel-icon" class:error={isError} class:success={!isError && outputItems.length > 0}>
          {#if isError}
            <svg viewBox="0 0 24 24" fill="currentColor">
              <path d="M12 2C6.48 2 2 6.48 2 12s4.48 10 10 10 10-4.48 10-10S17.52 2 12 2zm1 15h-2v-2h2v2zm0-4h-2V7h2v6z"/>
            </svg>
          {:else}
            <svg viewBox="0 0 24 24" fill="currentColor">
              <path d="M9 16.17L4.83 12l-1.42 1.41L9 19 21 7l-1.41-1.41z"/>
            </svg>
          {/if}
        </span>
        <label for="output">Output</label>
        {#if outputItems.length > 1}
          <span class="count">{outputItems.length} items</span>
        {/if}
      </div>
      <div class="output-list" class:error={isError}>
        {#each outputItems as item, i}
          <div class="output-item">
            <span class="item-index">{i + 1}</span>
            <pre><code>{@html isError ? item : highlightHtml(item)}</code></pre>
          </div>
        {/each}
      </div>
    </div>
  </main>

  <footer>
    <p>Inspired by <a href="https://regex101.com" target="_blank" rel="noopener">regex101</a></p>
  </footer>
</div>

<style>
  .playground {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
    padding: 1rem 1.5rem;
  }

  header {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    margin-bottom: 1rem;
    animation: fadeIn 0.6s ease-out;
  }

  header svg {
    width: 22px;
    height: 22px;
    color: var(--accent-primary);
    flex-shrink: 0;
  }

  header h1 {
    font-size: 1.1rem;
    font-weight: 600;
    background: linear-gradient(135deg, var(--text-primary) 0%, var(--accent-primary) 100%);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    background-clip: text;
    margin: 0;
  }

  .separator {
    color: var(--text-muted);
  }

  .tagline {
    color: var(--text-secondary);
    font-size: 0.85rem;
    margin: 0;
  }

  main {
    flex: 1;
    display: flex;
    flex-direction: column;
    gap: 1rem;
  }

  .panel {
    background: var(--bg-secondary);
    border: 1px solid var(--border-primary);
    border-radius: 8px;
    overflow: hidden;
    transition: border-color 0.2s ease;
  }

  .panel:focus-within {
    border-color: var(--border-focus);
    box-shadow: 0 0 0 3px var(--accent-glow);
  }

  .panel-header {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem 0.75rem;
    background: var(--bg-tertiary);
    border-bottom: 1px solid var(--border-primary);
  }

  .panel-icon {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 20px;
    height: 20px;
    color: var(--text-muted);
  }

  .panel-icon svg {
    width: 16px;
    height: 16px;
  }

  .panel-icon.error {
    color: var(--error);
  }

  .panel-icon.success {
    color: var(--success);
  }

  .panel-header label {
    font-size: 0.8rem;
    font-weight: 500;
    text-transform: uppercase;
    letter-spacing: 0.05em;
    color: var(--text-secondary);
  }

  .hint {
    margin-left: auto;
    font-size: 0.75rem;
    color: var(--text-muted);
    font-family: var(--font-mono);
  }

  .pretty-print-btn {
    margin-left: auto;
    display: flex;
    align-items: center;
    gap: 0.35rem;
    padding: 0.25rem 0.5rem;
    background: var(--bg-secondary);
    border: 1px solid var(--border-primary);
    border-radius: 4px;
    color: var(--text-secondary);
    font-family: var(--font-sans);
    font-size: 0.7rem;
    cursor: pointer;
    transition: all 0.15s ease;
  }

  .pretty-print-btn:hover {
    background: var(--bg-tertiary);
    color: var(--text-primary);
    border-color: var(--border-focus);
  }

  .pretty-print-btn svg {
    width: 14px;
    height: 14px;
  }

  .editor-container {
    position: relative;
    background: var(--bg-input);
    overflow: hidden;
  }

  .editor-highlight {
    position: absolute;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    margin: 0;
    padding: 0.75rem;
    font-family: var(--font-mono);
    font-size: 0.75rem;
    line-height: 1.5;
    white-space: pre-wrap;
    word-wrap: break-word;
    overflow: auto;
    pointer-events: none;
    color: var(--text-primary);
    background: transparent;
  }

  .editor-highlight::-webkit-scrollbar {
    display: none;
  }

  .editor-highlight code {
    font-family: inherit;
    font-size: inherit;
    line-height: inherit;
    display: block;
  }

  textarea {
    position: relative;
    width: 100%;
    border: none;
    outline: none;
    resize: none;
    background: transparent;
    color: transparent;
    caret-color: var(--text-primary);
    font-family: var(--font-mono);
    font-size: 0.75rem;
    line-height: 1.5;
    padding: 0.75rem;
    overflow: auto;
  }

  .html-panel .editor-container {
    height: 180px;
  }

  .html-panel textarea {
    height: 100%;
  }

  .code-panel .editor-container {
    height: 80px;
  }

  .code-panel textarea {
    height: 100%;
  }

  /* Prism.js syntax highlighting - dark theme */
  :global(.token.tag) {
    color: #7ee787;
  }

  :global(.token.attr-name) {
    color: #79c0ff;
  }

  :global(.token.attr-value) {
    color: #a5d6ff;
  }

  :global(.token.punctuation) {
    color: #8b949e;
  }

  :global(.token.comment) {
    color: #8b949e;
    font-style: italic;
  }

  :global(.token.string) {
    color: #a5d6ff;
  }

  :global(.token.entity) {
    color: #ffa657;
  }

  .run-button {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 0.5rem;
    padding: 0.6rem 1.25rem;
    background: linear-gradient(135deg, var(--accent-secondary) 0%, #2ea043 100%);
    color: white;
    border: none;
    border-radius: 6px;
    font-family: var(--font-sans);
    font-size: 0.9rem;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.2s ease;
    box-shadow: 0 2px 8px rgba(35, 134, 54, 0.3);
  }

  .run-button:hover {
    transform: translateY(-1px);
    box-shadow: 0 4px 12px rgba(35, 134, 54, 0.4);
  }

  .run-button:active {
    transform: translateY(0);
  }

  .run-button svg {
    width: 16px;
    height: 16px;
  }

  .run-button kbd {
    padding: 0.2rem 0.4rem;
    background: rgba(255, 255, 255, 0.15);
    border-radius: 4px;
    font-family: var(--font-mono);
    font-size: 0.75rem;
    margin-left: 0.25rem;
  }

  .count {
    margin-left: auto;
    font-size: 0.75rem;
    color: var(--text-muted);
    font-family: var(--font-mono);
  }

  .output-list {
    min-height: 120px;
    padding: 0.5rem;
    display: flex;
    flex-direction: column;
    gap: 0.5rem;
    background: var(--bg-input);
  }

  .output-list.error .output-item pre {
    color: var(--error);
  }

  .output-item {
    display: flex;
    gap: 0.5rem;
    background: var(--bg-tertiary);
    border: 1px solid var(--border-primary);
    border-radius: 6px;
    overflow: hidden;
  }

  .item-index {
    display: flex;
    align-items: center;
    justify-content: center;
    min-width: 2rem;
    padding: 0.5rem;
    background: var(--bg-secondary);
    color: var(--text-muted);
    font-family: var(--font-mono);
    font-size: 0.75rem;
    border-right: 1px solid var(--border-primary);
  }

  .output-item pre {
    flex: 1;
    margin: 0;
    padding: 0.5rem;
    font-family: var(--font-mono);
    font-size: 0.75rem;
    line-height: 1.5;
    white-space: pre-wrap;
    word-break: break-word;
    color: var(--text-primary);
    background: transparent;
  }

  footer {
    text-align: center;
    padding-top: 1rem;
    color: var(--text-muted);
    font-size: 0.8rem;
  }

  footer a {
    color: var(--accent-primary);
    text-decoration: none;
    border-bottom: 1px solid transparent;
    transition: border-color 0.2s ease;
  }

  footer a:hover {
    border-bottom-color: var(--accent-primary);
  }

  @keyframes fadeIn {
    from {
      opacity: 0;
      transform: translateY(-10px);
    }
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  @media (max-width: 640px) {
    .playground {
      padding: 0.75rem;
    }

    .tagline, .separator {
      display: none;
    }

    .hint {
      display: none;
    }
  }
</style>

