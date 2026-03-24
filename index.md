---
      layout: default
      title: "Shared Project Documents"
      description: "A small link-only portal for the project documents that live in `_docs/`."
      ---

      This site is meant for direct sharing by URL. It is intentionally configured with `noindex` meta tags and a restrictive `robots.txt`, but anyone who has a page link can still open it.

      ## Available Documents

      <ul class="doc-list">
        <li>
  <a href="{{ '/documents/master-architecture/' | relative_url }}">Master Architecture v4</a>
  <p>Canonical architecture for the icon store operating model.</p>
</li>
<li>
  <a href="{{ '/documents/pipeline-overview/' | relative_url }}">Pipeline Overview</a>
  <p>Operational summary of the end-to-end catalog and publishing pipeline.</p>
</li>
<li>
  <a href="{{ '/documents/spec-1-catalog-data-model/' | relative_url }}">Spec 1: Catalog Data Model</a>
  <p>PostgreSQL catalog schema, field map, and canonical data model.</p>
</li>
<li>
  <a href="{{ '/documents/spec-2-image-pipeline/' | relative_url }}">Spec 2: Image Pipeline</a>
  <p>Image normalization, derivatives, storage, and QA workflow.</p>
</li>
<li>
  <a href="{{ '/documents/spec-3-content-generation/' | relative_url }}">Spec 3: Content Generation</a>
  <p>Claude-driven content generation, review flow, and output contract.</p>
</li>
<li>
  <a href="{{ '/documents/spec-4-amazon-launch/' | relative_url }}">Spec 4: Amazon Launch</a>
  <p>Amazon launch playbook, GTIN exemption path, and SP-API operations.</p>
</li>
      </ul>
