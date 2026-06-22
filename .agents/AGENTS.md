# Workspace Rules for Knowledge Base

## Custom Command Triggers

### `/study-guide <topic>` or `/learn-topic <topic>`
* **Trigger**: When the user enters a message starting with `/study-guide` or `/learn-topic` followed by a topic name.
* **Action**: Activate the **[study_guide_generator](file:///D:/KnowladgeBase/.agents/skills/study_guide_generator/SKILL.md)** skill. 
* **Behavior**:
  1. Immediately search the web for the requested topic.
  2. Gather, filter, and structure the latest information combined with classic literature references.
  3. Generate a comprehensive, large-scale study guide markdown document containing Mermaid diagrams, practical examples, and case studies.
  4. Save this generated document directly to the `Kubernetes/` directory (or appropriate subfolder in the workspace) and link it in your response.
