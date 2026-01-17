```
(async () => {
  const sleep = ms => new Promise(r => setTimeout(r, ms));

  const visited = new Set();
  const pages = {};

  const ROOT_SELECTOR = '[data-content-editable-root="true"]';

  function getPageIdFromUrl(url) {
    const match = url.match(/[0-9a-f]{32}|[0-9a-f-]{36}/i);
    return match ? match[0] : null;
  }

  function getCurrentPageId() {
    return getPageIdFromUrl(location.href);
  }

  async function forceLoad() {
    let last = 0;
    for (let i = 0; i < 8; i++) {
      window.scrollTo(0, document.body.scrollHeight);
      await sleep(500);
      const h = document.body.scrollHeight;
      if (h === last) break;
      last = h;
    }
    window.scrollTo(0, 0);
  }

  function extractPageData() {
    const root = document.querySelector(ROOT_SELECTOR);
    if (!root) return null;

    const title =
      root.querySelector(".notion-page-block h1")?.innerText?.trim() || "";

    const contentRoot = root.querySelector(".notion-page-content");
    const text = contentRoot ? contentRoot.innerText.trim() : "";

    const links = Array.from(
      contentRoot.querySelectorAll(
        '.notion-page-block a[href^="/"]'
      )
    )
      .map(a => new URL(a.getAttribute("href"), location.origin).href)
      .filter(Boolean);

    return { title, text, links };
  }

  async function navigateTo(url) {
    history.pushState({}, "", url);
    window.dispatchEvent(new PopStateEvent("popstate"));
    await sleep(900);
    await forceLoad();
  }

  async function crawl(url) {
    const pageId = getPageIdFromUrl(url);
    if (!pageId || visited.has(pageId)) return;

    visited.add(pageId);
    console.log("📄 Crawling:", url);

    await navigateTo(url);

    const data = extractPageData();
    if (!data) return;

    pages[pageId] = {
      url,
      title: data.title,
      content: data.text
    };

    for (const link of data.links) {
      const childId = getPageIdFromUrl(link);
      if (childId && !visited.has(childId)) {
        await crawl(link);
      }
    }
  }

  await crawl(location.href);

  const blob = new Blob(
    [JSON.stringify(pages, null, 2)],
    { type: "application/json" }
  );

  const a = document.createElement("a");
  a.href = URL.createObjectURL(blob);
  a.download = "notion_docs_export.json";
  document.body.appendChild(a);
  a.click();
  a.remove();

  console.log("✅ Done. Pages captured:", Object.keys(pages).length);
})();

```
