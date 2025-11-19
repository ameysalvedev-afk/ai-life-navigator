# ai-life-navigator
Python + OpenAI-style agent API / pseudo-agent prompts

#!/usr/bin/env python3
"""
ai_life_navigator_cli.py
Minimal CLI prototype for AI Life Navigator.
- Has Controller + 4 micro-agents (prompts from the library)
- Uses a simulated LLM by default (no API key needed)
- To use a real OpenAI-style API, set OPENAI_API_KEY and uncomment the call_llm_with_api function
"""

import json
import os
from datetime import datetime, timedelta

MEMORY_FILE = "memory.json"

# ---------------------------
# Utilities: memory handling
# ---------------------------
def load_memory():
    if os.path.exists(MEMORY_FILE):
        with open(MEMORY_FILE, "r") as f:
            return json.load(f)
    # default empty memory
    mem = {
        "user_id": "user123",
        "preferences": {},
        "profiles": {},
        "streaks": {},
        "mood_history": [],
        "plans": {}
    }
    save_memory(mem)
    return mem

def save_memory(mem):
    with open(MEMORY_FILE, "w") as f:
        json.dump(mem, f, indent=2)

# ---------------------------
# Simple intent detection
# ---------------------------
def detect_intent(user_input: str):
    u = user_input.lower()
    if "plan" in u or "study" in u:
        return "plan_study"
    if "missed" in u or "replan" in u:
        return "update_plan"
    if "mood" in u or "stressed" in u or "tired" in u:
        return "check_mood"
    if "todo" in u or "task" in u or "add task" in u:
        return "todo_add"
    if "career" in u or "learn" in u and ("ml" in u or "ai" in u):
        return "career_tip"
    if "export memory" in u:
        return "export_memory"
    return "clarify"

# ---------------------------
# Simulated LLM responses (for testing)
# ---------------------------
def simulated_study_planner(input_json):
    subjects = input_json.get("subjects", [])
    days_left = input_json.get("days_left", 7)
    daily_hours = input_json.get("daily_hours", 2)
    minutes_per_day = int(daily_hours * 60)
    plan = []
    # naive round-robin prioritising hard topics first
    topic_queue = []
    for s in subjects:
        for t in s.get("topics", []):
            priority = 2 if s.get("difficulty") == "hard" else (1 if s.get("difficulty")=="medium" else 0)
            topic_queue.append({"subject": s["name"], "topic": t, "priority": priority})
    # sort by priority desc
    topic_queue.sort(key=lambda x: -x["priority"])
    i = 0
    for day in range(1, days_left+1):
        minutes_left = minutes_per_day
        tasks = []
        while minutes_left > 20 and i < len(topic_queue):
            take = min(60, minutes_left)  # chunk up to 60 min
            tq = topic_queue[i]
            tasks.append({"subject": tq["subject"], "topic": tq["topic"], "minutes": take})
            minutes_left -= take
            i += 1
        plan.append({"day": day, "tasks": tasks})
    return {
        "type":"study_plan",
        "summary": f"{days_left}-day plan, {daily_hours}h/day, {len(subjects)} subjects",
        "plan": plan,
        "notes":"If you miss a day, tell the agent 'I missed day X' and it will reassign topics."
    }

def simulated_productivity_agent(input_json):
    title = input_json.get("task_title","Untitled Task")
    effort = input_json.get("effort","medium")
    est = 60 if effort=="high" else (30 if effort=="medium" else 15)
    subtasks = [f"{title} - step {i+1}" for i in range(3)]
    return {"type":"todo","task":{"title":title,"subtasks":subtasks,"priority":"high" if effort=="high" else "med","estimated_minutes":est},"schedule_suggestion":"Do it in your next 60-min slot"}

def simulated_wellness_agent(input_json):
    mood = input_json.get("mood","neutral")
    stress = int(input_json.get("stress_level",3))
    activities = [
        {"name":"5-min breathing","duration_minutes":5,"instructions":"Sit upright, inhale 4s, hold 4s, exhale 6s"},
        {"name":"short walk","duration_minutes":10,"instructions":"Walk around, focus on senses"},
        {"name":"micro-break","duration_minutes":5,"instructions":"Stand, stretch, look away from screen"}
    ]
    motiv = "You've got this — one small step counts."
    if stress >= 7:
        activities.append({"name":"talk-to-friend","duration_minutes":0,"instructions":"Consider speaking to a trusted person or counselor."})
        motiv = "High stress detected — take a short break and seek support if needed."
    return {"type":"wellness","activities":activities,"motivational_line":motiv}

def simulated_career_agent(input_json):
    level = input_json.get("level","beginner")
    time = int(input_json.get("time_per_day_minutes",30))
    daily_plan = [{"day":i+1,"micro_learning":"implement a small algorithm","resource_type":"practice","time_minutes":time} for i in range(7)]
    return {"type":"career_tip","daily_plan":daily_plan,"one_week_goal":"Implement a simple ML pipeline and train on a small dataset"}

# ---------------------------
# Controller: route to micro-agents
# ---------------------------
def controller_route(intent, user_payload):
    if intent == "plan_study":
        return {"route":"study_planner","agent_input":user_payload}
    if intent == "update_plan":
        return {"route":"study_planner","agent_input":user_payload}
    if intent == "check_mood":
        return {"route":"wellness","agent_input":user_payload}
    if intent == "todo_add":
        return {"route":"productivity","agent_input":user_payload}
    if intent == "career_tip":
        return {"route":"career","agent_input":user_payload}
    if intent == "export_memory":
        return {"route":"export","agent_input":user_payload}
    return {"route":"clarify","agent_input":{"question":"I didn't understand — do you want a study plan, mood check, todo, or career tips?"}}

# ---------------------------
# Call micro-agents (simulated or real)
# ---------------------------
def call_agent(route, agent_input):
    if route == "study_planner":
        return simulated_study_planner(agent_input)
    if route == "productivity":
        return simulated_productivity_agent(agent_input)
    if route == "wellness":
        return simulated_wellness_agent(agent_input)
    if route == "career":
        return simulated_career_agent(agent_input)
    if route == "export":
        mem = load_memory()
        return {"type":"memory_export","memory": mem}
    return {"type":"clarify","question":"Please re-state your request"}

# ---------------------------
# CLI interaction
# ---------------------------
def main():
    print("AI Life Navigator — CLI Prototype\n(Type 'exit' to quit.)")
    memory = load_memory()
    while True:
        user = input("\nYou: ").strip()
        if user.lower() in ("exit","quit"):
            print("Good luck! Memory saved.")
            break
        intent = detect_intent(user)
        print(f"[debug] detected intent: {intent}")
        # Quick structured prompt builder for common intents
        if intent == "plan_study":
            # gather quick fields from user interactively
            subjects_raw = input("Enter subjects (comma-separated, with difficulty e.g. Maths:hard,Physics:medium): ").strip()
            subjects = []
            for part in subjects_raw.split(","):
                if ":" in part:
                    name, diff = [x.strip() for x in part.split(":",1)]
                else:
                    name, diff = part.strip(), "medium"
                topics = input(f"List 3 topics for {name} (comma-separated): ").strip().split(",")
                topics = [t.strip() for t in topics if t.strip()]
                subjects.append({"name":name,"topics":topics,"difficulty":diff})
            days_left = int(input("Days left until exam (int): ").strip() or "7")
            daily_hours = float(input("Daily hours available (e.g., 2.5): ").strip() or "2")
            payload = {"subjects":subjects,"days_left":days_left,"daily_hours":daily_hours}
        elif intent == "check_mood":
            mood = input("How do you feel? (stressed/tired/good/neutral/sad): ").strip()
            stress = int(input("Stress level 0-10: ").strip() or "5")
            payload = {"mood":mood,"stress_level":stress}
            # update memory
            memory.setdefault("mood_history",[]).append({"date": datetime.now().strftime("%Y-%m-%d"), "mood":mood, "stress_level":stress})
            save_memory(memory)
        elif intent == "todo_add":
            title = input("Task title: ").strip()
            deadline = input("Deadline (YYYY-MM-DD) or enter for none: ").strip() or None
            effort = input("Effort (low/medium/high): ").strip() or "medium"
            payload = {"task_title":title,"task_deadline":deadline,"effort":effort}
        elif intent == "career_tip":
            level = input("Your level (beginner/intermediate): ").strip() or "beginner"
            interests = input("Interests (comma-separated): ").strip().split(",")
            time_per_day = int(input("Minutes per day you can spend: ").strip() or "30")
            payload = {"level":level,"interests":interests,"time_per_day_minutes":time_per_day}
        elif intent == "export_memory":
            payload = {}
        else:
            # fallback: ask a quick clarifying Q
            print("I didn't understand. Do you want: (1) study plan (2) mood check (3) add todo (4) career tips ?")
            continue

        route_obj = controller_route(intent, payload)
        print(f"[debug] controller routing to: {route_obj['route']}")
        agent_out = call_agent(route_obj["route"], route_obj["agent_input"])
        # pretty print result
        print("\n--- Agent Response ---")
        print(json.dumps(agent_out, indent=2))
        print("----------------------")
        # example memory update: save last plan summary
        if agent_out.get("type") == "study_plan":
            memory.setdefault("plans",{})["last_plan_generated"] = {"date": datetime.now().strftime("%Y-%m-%d"), "summary": agent_out.get("summary")}
            save_memory(memory)

if __name__ == "__main__":
    main()
