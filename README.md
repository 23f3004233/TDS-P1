# TDS-P1

This repository holds the code for the **LLM Code Deployment** project, a core component of the Tools in Data Science course (Project 1) at IIT Madras, Online Data Science and Applications (Diploma Level).

-----

## **Project Overview**

The primary goal of this project is to create an application that can **programmatically build, deploy, and update another application** using the Gemini API. This is achieved by hosting an API endpoint that listens for task requests, utilizes an LLM to generate the required code, pushes it to a new GitHub repository with Pages enabled, and then notifies an evaluation server.

The application acts as a fully automated software development pipeline. It receives an application brief, generates the complete codebase (including `README.md` and `LICENSE`), deploys it to a separate, newly-created GitHub repository (GitHub B), and handles subsequent revision requests.

### **How It Works**

1.  **Request Reception:** The main application (in this repo, GitHub A) exposes an API endpoint to receive a JSON task request.
2.  **Code Generation:** The request brief is fed to the **Gemini API** to generate the minimal required application code.
3.  **Deployment:** The generated code is pushed to a **new, unique public GitHub repository** (GitHub B) using the GitHub API/CLI.
4.  **GitHub Pages:** **GitHub Pages** is enabled on the new repository (GitHub B) to make the generated application live.
5.  **Evaluation Ping:** The repository metadata (URLs, commit SHA) is sent to a specified `evaluation_url`.
6.  **Revision:** The application also handles follow-up requests (`round: 2`) to update the deployed code based on new requirements.

-----

## **Live Application Demo**

You can interact with the live deployed API endpoint for this project here:

https://devodita-tdsp.hf.space/ready

-----

## **Repository Structure**

This repository (GitHub A) contains the core logic for the build, deployment, and update process.

```
.
├── README.md           # This file
├── main.py             # Contains the core API logic (Flask/FastAPI)
├── requirements.txt    # Lists all necessary Python dependencies
└── Dockerfile          # Defines the environment for containerized deployment
```

### **Key Technologies**

  * **Core Logic:** Python
  * **LLM Integration:** **Gemini API** for code generation.
  * **Deployment:** **GitHub API/CLI** for repository creation and pushing.
  * **Deployment Hosting:** This application is hosted on **Hugging Face Spaces**.

-----

## **Setup and Installation**

### **Prerequisites**

You will need the following accounts and credentials:

1.  **Google AI Studio API Key:** For accessing the Gemini API.
2.  **GitHub Personal Access Token (PAT):** With scopes for creating and pushing to public repositories. This token must be kept secret.

### **Local Setup**

1.  **Clone the Repository:**

    ```bash
    git clone https://github.com/your-username/your-repo-name.git
    cd your-repo-name
    ```

2.  **Install Dependencies:**

    ```bash
    pip install -r requirements.txt
    ```

3.  **Set Environment Variables:**
    The application relies on environment variables for sensitive credentials:

    | Variable Name | Description |
    | :--- | :--- |
    | `GEMINI_API_KEY` | Your Google AI Studio API Key. |
    | `GITHUB_PAT` | Your GitHub Personal Access Token. |
    | `STUDENT_EMAIL` | Your IITM student email (e.g., `ee23s000@smail.iitm.ac.in`). |
    | `EXPECTED_SECRET` | The shared secret string submitted in the Google Form. |

    You can set these in a `.env` file (and add `.env` to `.gitignore`) or directly in your shell:

    ```bash
    export GEMINI_API_KEY="AIza..."
    # ... and so on for other variables
    ```

4.  **Run the Application:**
    The `main.py` file exposes the API endpoint.

    ```bash
    python main.py
    # or use Gunicorn/Uvicorn for production-grade serving
    ```

### **Docker Deployment**

A `Dockerfile` is provided for containerizing the application.

1.  **Build the Docker Image:**
    ```bash
    docker build -t llm-code-deployer .
    ```
2.  **Run the Container:**
    Pass your environment variables during the run command.
    ```bash
    docker run -p 8000:8000 \
      -e GEMINI_API_KEY="AIza..." \
      -e GITHUB_PAT="ghp_..." \
      -e EXPECTED_SECRET="your-secret-key" \
      llm-code-deployer
    ```
    The API will be accessible at `http://localhost:8000`.

-----

## **Usage: API Endpoint**

The application exposes a single endpoint that accepts `POST` requests.

### **Endpoint URL**

The URL will vary depending on your hosting environment (e.g., `https://devodita-tdsp.hf.space/ready`).

### **Request Body**

The API expects a JSON payload matching the request structure defined by the instructors:

| Key | Type | Description |
| :--- | :--- | :--- |
| `email` | string | Student email ID. |
| `secret` | string | Student-provided secret for verification. |
| `task` | string | Unique task ID (e.g., `captcha-solver-abc12`). |
| `round` | integer | Task round (1 for initial build, 2 for revision). |
| `nonce` | string | Nonce to be passed back to the evaluation URL. |
| `brief` | string | Application requirements. |
| `checks` | array | Evaluation criteria. |
| `evaluation_url` | string | The URL to POST the repo details after deployment. |
| `attachments` | array | Optional file attachments (e.g., `sample.png`) as data URIs. |

**Example `curl` Command:**

```bash
curl https://your-deployed-url/api-endpoint \
  -H "Content-Type: application/json" \
  -d '{
    "email": "...",
    "secret": "...",
    "task": "sum-of-sales-...",
    "round": 1,
    "nonce": "ab12-...",
    "brief": "Publish a single-page site...",
    "evaluation_url": "https://example.com/notify"
  }'
```

### **Response**

The endpoint is designed to return a simple **HTTP 200 OK** JSON response upon successful reception and verification, indicating the process has started:

```json
{
  "status": "received",
  "message": "Task received and deployment initiated."
}
```

-----

## **Code Explanation (Internal Logic)**

The `main.py` script orchestrates the entire process:

1.  **Secret Verification:** On receiving a `POST` request, it first verifies the `secret` in the payload against the `EXPECTED_SECRET` environment variable.
2.  **LLM Call:** It constructs a detailed prompt using the `brief`, `checks`, and `attachments`, and sends it to the **Gemini API**. The prompt asks for a complete, minimal HTML/JS application, a professional `README.md`, and an `LICENSE` file.
3.  **GitHub Repo Management:**
      * **New Repo Creation (Round 1):** Uses the `task` ID to generate a unique public repository name.
      * **File Commit:** Commits the generated files (HTML, `README.md`, `LICENSE`) to the repository.
      * **Revision (Round 2):** Fetches the existing code, applies the changes requested in the new `brief` (via a second LLM call), and pushes a new commit.
4.  **GitHub Pages Activation:** Ensures GitHub Pages is enabled on the `main` branch.
5.  **Evaluation Notification:** Constructs the final JSON payload with `repo_url`, `commit_sha`, and `pages_url`, and POSTs it to the provided `evaluation_url`, handling retries on error.
