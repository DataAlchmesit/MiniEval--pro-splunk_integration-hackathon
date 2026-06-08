# MiniEval--pro-splunk_integration-hackathon
MiniEval Pro - AI Hallucination Detection for splunk Security. Catches hallucinations in Splunk AI-generated security alerts before anlalysts waste time. 1,541 Hallucination caught. 200x cheaper than GPT-4.

## Setup Instructions

1. Clone the repository
2. Create virtual environment: `python -m venv gritvenv`
3. Activate: `gritvenv\Scripts\activate` (Windows)
4. Install dependencies: `pip install -r requirements.txt`
5. Copy `.env.example` to `.env` and add your Splunk HEC token
6. Run the pipeline: `python demo/run_demo.py`
7. Import dashboard XML into Splunk

## Dependencies

See `requirements.txt` for full list.

## Architecture

See `architecture_diagram.png` for system architecture.

## Results

- 2,130 evaluations
- 1,541 hallucinations caught
- 1,065 analyst hours saved
- 200x cheaper than GPT-4

## License

MIT
