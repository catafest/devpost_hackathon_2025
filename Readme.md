# YouTube Data Analysis Agent Workflow - by catafest

This project demonstrates a simple agent-based system in a Google Colab notebook to interact with the YouTube Data API, analyze video information, generate insights, and produce a report. While not using the official Google Agent SDK, it follows a similar pattern of defining specialized agents orchestrated to perform a task.

## Code Explanation

The core functionality is implemented across two main code cells.

### Cell - project source code: Agent and Orchestrator Definitions

This cell defines the Python classes that represent the different agents and the orchestrator.

#### Key Classes:

*   **`TitleSuggestionAgent`**: (Currently a placeholder) This agent is intended to suggest optimized video titles based on a topic. It has an `execute` method that takes input data (a dictionary) and returns a dictionary with suggested titles.
*   **`DescriptionGeneratorAgent`**: (Currently a placeholder) This agent is intended to generate video descriptions. It has an `execute` method that takes input data (a dictionary, potentially including titles) and returns a dictionary with a generated description.
*   **`TagOptimizerAgent`**: This agent interacts directly with the YouTube Data API.
    *   **`__init__(self, api_key, orchestrator=None)`**: Initializes the agent with your YouTube API key and an optional reference to the `Orchestrator` for bidirectional communication.
    *   **`execute(self, input_data: dict, callback=None)`**: Performs a YouTube search based on `keywords` from `input_data`. It fetches video details including snippet, statistics, and topic details. It includes an optional `callback` function that is called with the results upon completion. It also demonstrates bidirectional communication by calling `self.orchestrator.receive_agent_output`.
*   **`VideoAnalyzerAgent`**: This agent analyzes the data received from the `TagOptimizerAgent`.
    *   **`execute(self, input_data: dict, callback=None)`**: Takes `videos` data from `input_data`. It calculates the number of videos found, identifies the most common tags, calculates average statistics (views, likes, comments), finds the video with the highest views, identifies the most recent video, and analyzes channel distribution. It also supports an optional `callback`.
*   **`InsightAgent`**: This agent processes the analysis results from the `VideoAnalyzerAgent` to generate user-friendly insights.
    *   **`execute(self, input_data: dict, callback=None)`**: Takes `analysis_results` from `input_data`. It generates a list of insights based on the statistics and findings from the analysis. It supports an optional `callback`.
*   **`ReportGeneratorAgent`**: This agent formats the analysis results and insights into a readable report.
    *   **`execute(self, input_data: dict, callback=None)`**: Takes `report_data` (analysis results) and `insights` from `input_data`. It constructs a formatted report string that includes the key insights and detailed analysis findings. It supports an optional `callback`.
*   **`Orchestrator`**: This class manages the workflow and communication between the different agents. It acts as the central controller.
    *   **`__init__(self, api_key, output_widget)`**: Initializes the orchestrator with the YouTube API key, a reference to the `ipywidgets.Output` widget for displaying output, and instantiates the different agent classes. It also sets up a mechanism to capture all printed output.
    *   **`_capture_output(self, text)`**: An internal helper method to capture printed text and also display it in the output widget.
    *   **`run_workflow(self, keywords)`**: Starts the entire process. It redirects `print` to capture output, clears previous output, stores the keywords, and triggers the `TagOptimizerAgent`. Includes error handling and ensures `print` is restored.
    *   **`handle_tag_agent_results(self, results)`**: A callback method triggered by `TagOptimizerAgent`. It receives the search results, stores the video data, and triggers the `VideoAnalyzerAgent`. Includes error handling and print restoration.
    *   **`handle_analyzer_agent_results(self, results)`**: A callback method triggered by `VideoAnalyzerAgent`. It receives the analysis results, updates the stored results, and triggers the `InsightAgent`. Includes error handling and print restoration.
    *   **`handle_insight_agent_results(self, results)`**: A callback method triggered by `InsightAgent`. It receives the generated insights, stores them, and triggers the `ReportGeneratorAgent`. Includes error handling and print restoration.
    *   **`handle_report_agent_results(self, results)`**: A callback method triggered by `ReportGeneratorAgent`. It receives the final report, displays it in the output widget, and saves the entire captured workflow output to a text file with a timestamp and keywords in the filename. Includes error handling and ensures `print` is restored at the very end of the workflow.
    *   **`display_report(self, report_data)`**: An internal helper method to format and print the final report to the captured output (which is also displayed in the widget).
    *   **`receive_agent_output(self, agent, output)`**: A method for agents to send information back to the orchestrator (demonstrating bidirectional communication).

### Cell - user interface : GUI Setup and Workflow Execution

This cell sets up the interactive user interface and initiates the agent workflow when the button is clicked.

#### Key Components:

*   **`ipywidgets`**: Used to create interactive elements like a text input (`Text`), a button (`Button`), and an output area (`Output`).
*   **GUI Layout**: `VBox` is used to arrange the widgets vertically.
*   **API Key Loading**: Loads the `YOUTUBE_API_KEY` from the `.env` file.
*   **Orchestrator Instantiation**: Creates an instance of the `Orchestrator` class, passing the API key and the output widget.
*   **`on_button_click(button)` function**: This function is triggered when the search button is clicked. It clears the output area, retrieves the keywords from the input box, and calls the `orchestrator.run_workflow()` method with the keywords, starting the entire process.
*   **Button Event Handling**: `search_button.on_click(on_button_click)` attaches the `on_button_click` function to the button's click event.

## Workflow Functionality

The system operates as follows:

1.  The user enters keywords into the text box in the GUI and clicks the "Search YouTube" button.
2.  The `on_button_click` function is triggered, clears the previous output, and calls the `Orchestrator`'s `run_workflow` method.
3.  The `Orchestrator` initiates the workflow by calling the `TagOptimizerAgent.execute` method, passing the keywords and a callback (`handle_tag_agent_results`).
4.  The `TagOptimizerAgent` performs a YouTube search, fetches video details (including statistics and tags), and calls the `Orchestrator`'s `handle_tag_agent_results` callback with the video data. It can also send bidirectional messages to the orchestrator's `receive_agent_output`.
5.  The `handle_tag_agent_results` method in the `Orchestrator` receives the video data and triggers the `VideoAnalyzerAgent.execute` method, passing the video data and a callback (`handle_analyzer_agent_results`).
6.  The `VideoAnalyzerAgent` analyzes the video data (counts, common tags, averages, etc.) and calls the `Orchestrator`'s `handle_analyzer_agent_results` callback with the analysis results.
7.  The `handle_analyzer_agent_results` method receives the analysis results and triggers the `InsightAgent.execute` method, passing the analysis results and a callback (`handle_insight_agent_results`).
8.  The `InsightAgent` generates insights based on the analysis and calls the `Orchestrator`'s `handle_insight_agent_results` callback with the insights.
9.  The `handle_insight_agent_results` method receives the insights and triggers the `ReportGeneratorAgent.execute` method, passing the analysis results and the insights, along with a callback (`handle_report_agent_results`).
10. The `ReportGeneratorAgent` formats the analysis results and insights into a report and calls the `Orchestrator`'s `handle_report_agent_results` callback with the final report.
11. The `handle_report_agent_results` method receives the report, displays it in the GUI's output area, and saves the entire captured workflow output (including all intermediate messages) to a text file named with a timestamp and the original keywords.

This orchestrated approach allows for modular development, where each agent has a specific responsibility, and the orchestrator manages the flow of data and control between them.