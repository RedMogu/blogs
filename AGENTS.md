# Engineering Blog Repository

## 📂 Repository Structure
- `/raw/`: Initial brain-dumps, architectural outlines, and AI-assisted drafts. **Draft here first.**
- `/posts/`: Finalized, "De-AI" cleaned articles. Source of truth for production.
- `/assets/`: Shared images and media.

## 🛠 Content Workflow
1. **Naming**: Use `YYYY-MM-DD-kebab-case-title.md` format.
2. **Drafting**: Create new material in `/raw/`.
3. **Format**: Posts must be bilingual (English + Chinese).
    - Top: Language toggle links.
    - Meta: **Date** and **Author** (Limina Engineering Team).
    - Body: Full English version, `---` separator, then full Chinese version.
4. **Cleaning**: Drafts must pass through the **"De-AI" (去AI化)** LangGraph workflow (located in the `coreos` repository) to strip formulaic LLM styles before moving to `/posts/`.
5. **Integration**: The `liminalabs-web` project fetches posts from `/posts/` (locally or via GitHub API).

## 🎨 Design System (xAI)
Adhere strictly to `DESIGN.md` for any UI/asset generation:
- **Theme**: Pure dark (`#1f2228`), pure white text (`#ffffff`). No shadows, no gradients.
- **Typography**: `GeistMono` (display/buttons), `universalSans` (body).
- **Brutalism**: Sharp corners (0px radius) default. 
- **Interaction**: Dim elements to 0.5 opacity on hover (reverse of standard convention).

## 🧠 Philosophy
- **Deterministic > Autonomous**: Prefer DAGs (LangGraph) and SOPs over letting LLMs "figure it out".
- **JIT Context**: Use Just-In-Time retrieval for cold data; avoid O(N²) global rewrites.
- **Production Reality**: Value battle-tested architectures over academic elegance.
