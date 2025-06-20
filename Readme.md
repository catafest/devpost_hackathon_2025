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

See the source code with all of these :
      
```
import os
from googleapiclient.discovery import build

from dotenv import load_dotenv

#
from collections import Counter
from datetime import datetime

from dotenv import load_dotenv

#
from ipywidgets import Text, Button, VBox, Output


# from googleapiclient.discovery import build
# from collections import Counter
class TitleSuggestionAgent:
    def execute(self, input_data: dict) -> dict:
        """Suggests optimized video titles."""
        video_topic = input_data.get("video_topic", "default topic")
        # Simulated LLM response (replace with actual LLM if available)
        titles = f"1. {video_topic}: Beginner's Guide\n2. Master {video_topic} in 2025\n3. Top {video_topic} Tips"
        return {"status": "success", "titles": titles}

class DescriptionGeneratorAgent:
    def execute(self, input_data: dict) -> dict:
        """Generates video description."""
        titles = input_data.get("titles", "")
        description = f"Welcome to our {titles.split(':')[0]} video! Learn key insights and tips.\nSubscribe for more!"
        return {"status": "success", "description": description}

class TagOptimizerAgent:
    def __init__(self, api_key, orchestrator=None):
        """Initializes the agent with the YouTube API key and an optional orchestrator."""
        self.api_key = api_key
        self.orchestrator = orchestrator # To enable communication with the orchestrator

    def execute(self, input_data: dict, callback=None) -> dict:
        """
        Performs a YouTube search based on keywords and returns video information,
        including statistics and other details.
        Can trigger a callback function with the results.
        """
        print("TagOptimizerAgent: Executing...")
        youtube = build("youtube", "v3", developerKey=self.api_key)
        keywords = input_data.get("keywords", "default topic")

        try:
            # Step 1: Perform the initial search
            search_request = youtube.search().list(
                q=keywords,
                part="snippet",
                maxResults=20, # Increased the number of results
                # maxResults=5,
                type="video"
            )
            search_response = search_request.execute()

            video_ids = []
            for item in search_response.get("items", []):
                if "videoId" in item["id"]:
                    video_ids.append(item["id"]["videoId"])

            videos = []
            if video_ids:
                # Step 2: Fetch video details including statistics and topicDetails
                video_details_request = youtube.videos().list(
                    part="snippet,statistics,topicDetails",
                    id=",".join(video_ids)
                )
                video_details_response = video_details_request.execute()

                # Step 3: Process the detailed video information
                for item in video_details_response.get("items", []):
                    video_info = {
                        "title": item["snippet"]["title"],
                        "url": f"https://www.youtube.com/watch?v={item['id']}",
                        "tags": item["snippet"].get("tags", []), # Handle missing tags
                        "channelTitle": item["snippet"].get("channelTitle", "N/A"),
                        "publishedAt": item["snippet"].get("publishedAt", "N/A"),
                        "viewCount": int(item["statistics"].get("viewCount", 0)), # Handle missing statistics
                        "likeCount": int(item["statistics"].get("likeCount", 0)),
                        "commentCount": int(item["statistics"].get("commentCount", 0)),
                        "topicDetails": item.get("topicDetails", {}) # Include topic details
                    }
                    videos.append(video_info)

            result = {"status": "success", "videos": videos}

            # Call the callback function if provided
            if callback:
                print("TagOptimizerAgent: Calling callback...")
                callback(result)

            # Example of sending data back to the orchestrator (bidirectional)
            if self.orchestrator:
                print("TagOptimizerAgent: Sending result to orchestrator...")
                self.orchestrator.receive_agent_output(self, result)

            return result

        except Exception as e:
            print(f"TagOptimizerAgent Error: {e}")
            error_result = {"status": "error", "message": str(e)}

            # Call callback with error if provided
            if callback:
                print("TagOptimizerAgent: Calling error callback...")
                callback(error_result)

            # Example of sending error back to the orchestrator
            if self.orchestrator:
                print("TagOptimizerAgent: Sending error to orchestrator...")
                self.orchestrator.receive_agent_output(self, error_result)

            return error_result
#
class VideoAnalyzerAgent:
    def execute(self, input_data: dict, callback=None) -> dict:
        """
        Analyzes video data including statistics and details,
        and can trigger a callback with the analysis results.
        """
        print("VideoAnalyzerAgent: Executing...")
        videos = input_data.get("videos", [])
        analysis_results = {}

        if videos:
            analysis_results["num_videos_found"] = len(videos)

            # Analyze tags
            all_tags = [tag for video in videos for tag in video.get("tags", [])]
            if all_tags:
                tag_counts = Counter(all_tags)
                most_common_tags = tag_counts.most_common(5) # Get top 5 most common tags
                analysis_results["most_common_tags"] = most_common_tags
            else:
                analysis_results["most_common_tags"] = []

            # Analyze statistics
            total_view_count = sum(video.get("viewCount", 0) for video in videos)
            total_like_count = sum(video.get("likeCount", 0) for video in videos)
            total_comment_count = sum(video.get("commentCount", 0) for video in videos)

            analysis_results["average_view_count"] = total_view_count / len(videos) if len(videos) > 0 else 0
            analysis_results["average_like_count"] = total_like_count / len(videos) if len(videos) > 0 else 0
            analysis_results["average_comment_count"] = total_comment_count / len(videos) if len(videos) > 0 else 0

            # Find video with highest view count
            highest_view_video = max(videos, key=lambda x: x.get("viewCount", 0), default=None)
            if highest_view_video:
                analysis_results["highest_view_video"] = {
                    "title": highest_view_video.get("title", "N/A"),
                    "viewCount": highest_view_video.get("viewCount", 0),
                    "url": highest_view_video.get("url", "N/A")
                }
            else:
                analysis_results["highest_view_video"] = None

            # Analyze publication dates
            most_recent_video = None
            latest_published_date = None
            for video in videos:
                published_at_str = video.get("publishedAt")
                if published_at_str and published_at_str != "N/A":
                    try:
                        # Parse the date string (assuming ISO 8601 format from YouTube API)
                        published_date = datetime.fromisoformat(published_at_str.replace("Z", "+00:00"))
                        if most_recent_video is None or published_date > latest_published_date:
                            latest_published_date = published_date
                            most_recent_video = {
                                "title": video.get("title", "N/A"),
                                "publishedAt": published_at_str,
                                "url": video.get("url", "N/A")
                            }
                    except ValueError:
                        print(f"VideoAnalyzerAgent: Could not parse date: {published_at_str}")
                        pass # Ignore videos with unparseable dates

            analysis_results["most_recent_video"] = most_recent_video

            # Analyze channel distribution
            channel_counts = Counter(video.get("channelTitle", "Unknown Channel") for video in videos)
            analysis_results["channel_distribution"] = dict(channel_counts) # Convert Counter to dict for output


            analysis_results["status"] = "success"
        else:
            analysis_results["status"] = "success" # Still a success if no videos to analyze
            analysis_results["num_videos_found"] = 0
            analysis_results["most_common_tags"] = []
            analysis_results["average_view_count"] = 0
            analysis_results["average_like_count"] = 0
            analysis_results["average_comment_count"] = 0
            analysis_results["highest_view_video"] = None
            analysis_results["most_recent_video"] = None
            analysis_results["channel_distribution"] = {}


        print(f"VideoAnalyzerAgent: Analysis Results: {analysis_results}")

        if callback:
            print("VideoAnalyzerAgent: Calling callback...")
            callback(analysis_results)

        return analysis_results

#
class InsightAgent:
    def execute(self, input_data: dict, callback=None) -> dict:
        """Analyzes video analysis results and generates insights."""
        print("InsightAgent: Executing...")
        analysis_results = input_data.get("analysis_results", {})
        insights = []

        num_videos_found = analysis_results.get("num_videos_found", 0)
        if num_videos_found > 0:
            insights.append(f"Found {num_videos_found} videos related to the topic.")

            most_common_tags = analysis_results.get("most_common_tags", [])
            if most_common_tags:
                top_tag = most_common_tags[0][0]
                insights.append(f"The most common tag is '{top_tag}', suggesting its importance for discoverability.")

            avg_views = analysis_results.get("average_view_count", 0)
            insights.append(f"On average, videos in this topic have approximately {avg_views:,.0f} views.")

            highest_view_video = analysis_results.get("highest_view_video")
            if highest_view_video:
                insights.append(f"The video with the highest views is '{highest_view_video.get('title', 'N/A')}' with {highest_view_video.get('viewCount', 0):,.0f} views, indicating a potential trend or popular content style.")

            most_recent_video = analysis_results.get("most_recent_video")
            if most_recent_video:
                 insights.append(f"The most recent video found is '{most_recent_video.get('title', 'N/A')}', published on {most_recent_video.get('publishedAt', 'N/A')}, highlighting recent activity in the topic.")

            channel_distribution = analysis_results.get("channel_distribution", {})
            if channel_distribution:
                num_channels = len(channel_distribution)
                insights.append(f"The videos are spread across {num_channels} different channels.")
                if num_channels > 0:
                    most_active_channel = max(channel_distribution, key=channel_distribution.get)
                    insights.append(f"'{most_active_channel}' is the most active channel found in this search with {channel_distribution[most_active_channel]} video(s).")


        else:
            insights.append("No video data available for analysis.")

        result = {"status": "success", "insights": "\n".join(insights)}
        print(f"InsightAgent: Generated Insights:\n{result['insights']}")

        if callback:
            print("InsightAgent: Calling callback...")
            callback(result)

        return result

# Define the Orchestrator class
class Orchestrator:
    def __init__(self, api_key, output_widget):
        self.api_key = api_key
        self.output_widget = output_widget
        self.tag_agent = TagOptimizerAgent(api_key=self.api_key, orchestrator=self)
        self.analyzer_agent = VideoAnalyzerAgent()
        self.insight_agent = InsightAgent() # Initialize the InsightAgent
        self.report_agent = ReportGeneratorAgent()
        self.analysis_results = {} # To store results between agents
        self.insights = "" # To store insights from the InsightAgent
        self.current_keywords = "" # Store current keywords for filename
        self._captured_output = "" # Attribute to store captured output
        self._original_print = __builtins__.print # Store the original print function

    def _capture_output(self, text):
        """Captures output printed to the output widget."""
        self._captured_output += text
        # Call the original print to also display in the output widget
        self._original_print(text, end='')


    def run_workflow(self, keywords):
        # Redirect print to capture output and display in the output widget
        __builtins__.print = self._capture_output
        self._captured_output = "" # Clear captured output at the start of a new workflow

        try:
            print(f"Orchestrator: Starting workflow for keywords: {keywords}\n")
            self.current_keywords = keywords # Store keywords
            # Step 1: Trigger TagOptimizerAgent
            self.tag_agent.execute({"keywords": keywords}, callback=self.handle_tag_agent_results)
        except Exception as e:
            print(f"Orchestrator Error during workflow execution: {e}\n")
            # Ensure original print is restored even if an error occurs before the end of the workflow
            __builtins__.print = self._original_print
        # Note: The final print restore happens in handle_report_agent_results
        # after the report is displayed and saved.


    def handle_tag_agent_results(self, results):
        # Output is already being captured by _capture_output, no need to redirect here
        print("Orchestrator: Received results from TagOptimizerAgent.\n")
        try:
            if results["status"] == "success":
                # Ensure the enriched video data is stored
                self.analysis_results["videos"] = results["videos"]
                # Step 2: Trigger VideoAnalyzerAgent with the enriched videos
                self.analyzer_agent.execute({"videos": self.analysis_results["videos"]}, callback=self.handle_analyzer_agent_results)
            else:
                print(f"Orchestrator: TagOptimizerAgent failed: {results.get('message', 'Unknown error')}\n")
                self.display_report({"status": "error", "message": "Failed to retrieve video data."})
                # Restore print here on error
                __builtins__.print = self._original_print
        except Exception as e:
            print(f"Orchestrator Error in handle_tag_agent_results: {e}\n")
            # Restore print here on error
            __builtins__.print = self._original_print


    def handle_analyzer_agent_results(self, results):
        # Output is already being captured by _capture_output, no need to redirect here
        print("Orchestrator: Received results from VideoAnalyzerAgent.\n")
        try:
            if results["status"] == "success":
                self.analysis_results.update(results) # Add analysis results
                # Step 3: Trigger InsightAgent with the analysis results
                self.insight_agent.execute({"analysis_results": self.analysis_results}, callback=self.handle_insight_agent_results)
            else:
                print(f"Orchestrator: VideoAnalyzerAgent failed: {results.get('message', 'Unknown error')}\n")
                self.display_report({"status": "error", "message": "Failed to analyze video data."})
                # Restore print here on error
                __builtins__.print = self._original_print
        except Exception as e:
            print(f"Orchestrator Error in handle_analyzer_agent_results: {e}\n")
            # Restore print here on error
            __builtins__.print = self._original_print


    def handle_insight_agent_results(self, results):
        # Output is already being captured by _capture_output, no need to redirect here
        print("Orchestrator: Received results from InsightAgent.\n")
        try:
            if results["status"] == "success":
                self.insights = results.get("insights", "") # Store the insights
                # Step 4: Trigger ReportGeneratorAgent with analysis results and insights
                self.report_agent.execute({"report_data": self.analysis_results, "insights": self.insights}, callback=self.handle_report_agent_results)
            else:
                print(f"Orchestrator: InsightAgent failed: {results.get('message', 'Unknown error')}\n")
                # Proceed to report generation even if insight generation failed
                self.report_agent.execute({"report_data": self.analysis_results, "insights": "Could not generate insights."}, callback=self.handle_report_agent_results)
        except Exception as e:
            print(f"Orchestrator Error in handle_insight_agent_results: {e}\n")
            # Restore print here on error
            __builtins__.print = self._original_print


    def handle_report_agent_results(self, results):
        # Output is already being captured by _capture_output, no need to redirect here
        print("Orchestrator: Received results from ReportGeneratorAgent.\n")
        try:
            if results["status"] == "success":
                # Display the final report (this will also be captured)
                self.display_report(results)
                # Step 6: Save the captured output to a file
                report_content = self._captured_output # Get the entire captured output
                if report_content and self.current_keywords:
                    try:
                        # Create a filename with timestamp and keywords
                        now = datetime.now()
                        timestamp = now.strftime("%H%M_%d_%m_%Y")
                        safe_keywords = self.current_keywords.replace(" ", "_").replace("/", "_")
                        filename = f"yt__{timestamp}_{safe_keywords}.txt"
                        with open(filename, "w", encoding="utf-8") as f:
                            f.write(report_content)
                        print(f"Orchestrator: Full workflow output saved to {filename}\n")
                    except Exception as e:
                        print(f"Orchestrator: Error saving workflow output to file: {e}\n")

            else:
                print(f"Orchestrator: ReportGeneratorAgent failed: {results.get('message', 'Unknown error')}\n")

            # Clear analysis results and insights for the next run
            self.analysis_results = {}
            self.insights = ""
            self.current_keywords = "" # Clear keywords after use

        except Exception as e:
            print(f"Orchestrator Error in handle_report_agent_results: {e}\n")
        finally:
            # Restore original print function
            __builtins__.print = self._original_print


    def display_report(self, report_data):
        # This method now just prints to the captured output
        if report_data.get("status") == "success":
            print("\n--- FINAL REPORT ---\n")
            print(report_data.get("report", "No report generated."))
            print("--------------------\n")
        else:
            print("\n--- REPORT GENERATION FAILED ---")
            print(f"Error: {report_data.get('message', 'Unknown error')}")
            print("------------------------------\n")


    def receive_agent_output(self, agent, output):
        """Example of a bidirectional communication method for agents to call."""
        # This method now just prints to the captured output
        print(f"Orchestrator: Received bidirectional output from {type(agent).__name__}\n")
        # You could add logic here to handle specific outputs from agents
```
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
