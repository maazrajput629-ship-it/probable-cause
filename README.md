# probable-cause
#!/usr/bin/env python3
import sys
import os
import json
import urllib.request
import urllib.error

# ==============================================================================
# MIND TO MACHINE HACKATHON - SINGLE FILE REPOSITORY AGENT PLANNER
# ==============================================================================

EMBEDDED_MESSAGES = """1 | Dean's office, email | 9 Nov | Approved: a 4-hour hackathon for 150 students in teams of 3. The budget is Rs 650,000, all in. Ten staff will be on duty. Please keep a 15-minute prayer break in the schedule.
2 | Margalla Event Halls, email | 3 Nov | Room D seats 80 at Rs 40,000. Hall A seats 200 at Rs 120,000. Hall B seats 200 at Rs 150,000. Hall C seats 300 at Rs 220,000. Halls B and C have a standby generator.
3 | Caterer, WhatsApp | 20 Oct | Standard lunch box Rs 1,000 per head.
4 | Caterer, WhatsApp | 12 Nov | Salam bhai! Standard lunch box ab Rs 1,200 per head hoga, basic wala Rs 800. Delivery free hai.
5 | Judging coordinator, email | 11 Nov | 20 judges have confirmed. The honorarium is Rs 5,000 each. A judge needs 15 minutes per team and should cover no more than three teams.
6 | Finance, email | 10 Nov | Already ordered from the event budget: prizes Rs 100,000, backup internet Rs 20,000, printing Rs 10,000.
7 | Student society, WhatsApp | 13 Nov | Can we get a DJ for the closing? The quote is Rs 50,000.
8 | A judge, WhatsApp | 14 Nov | I may be 10 minutes late on the day. Traffic on Srinagar Highway.
"""

class TraceLogger:
    def __init__(self):
        self.calls = []
        self.call_counter = 0

    def log(self, area: str, model: str, input_text: str, output_text: str, prompt_tokens: int, completion_tokens: int):
        self.call_counter += 1
        entry = {
            "call_number": self.call_counter,
            "model": model,
            "area": area,
            "input": input_text,
            "output": output_text,
            "prompt_tokens": prompt_tokens,
            "completion_tokens": completion_tokens,
            "total_tokens": prompt_tokens + completion_tokens
        }
        self.calls.append(entry)

    def total_tokens(self) -> int:
        return sum(c["total_tokens"] for c in self.calls)

    def print_trace(self):
        print("\n" + "=" * 72)
        print(" EXECUTION TRACE")
        print("=" * 72)
        for c in self.calls:
            print(f"Call #{c['call_number']} | Model: {c['model']} | Area: {c['area']}")
            print(f"  Input : {c['input'].strip()}")
            print(f"  Output: {c['output'].strip()}")
            print(f"  Tokens: prompt={c['prompt_tokens']}, completion={c['completion_tokens']}, total={c['total_tokens']}")
            print("-" * 72)
        print(f"TOTAL TOKENS: {self.total_tokens()}")
        print("=" * 72 + "\n")


class LLMClient:
    def __init__(self):
        self.api_base = os.environ.get("LLM_API_BASE", "http://localhost:11434/v1").rstrip("/")
        self.api_key = os.environ.get("LLM_API_KEY", "ollama")
        self.model = os.environ.get("LLM_MODEL", "llama3.2:3b")

    def chat(self, system_prompt: str, user_prompt: str) -> tuple[str, int, int]:
        url = f"{self.api_base}/chat/completions"
        payload = {
            "model": self.model,
            "messages": [
                {"role": "system", "content": system_prompt},
                {"role": "user", "content": user_prompt}
            ],
            "temperature": 0.0
        }
        headers = {
            "Content-Type": "application/json",
            "Authorization": f"Bearer {self.api_key}"
        }
        req = urllib.request.Request(
            url,
            data=json.dumps(payload).encode("utf-8"),
            headers=headers,
            method="POST"
        )
        try:
            with urllib.request.urlopen(req, timeout=45) as resp:
                data = json.loads(resp.read().decode("utf-8"))
                content = data["choices"][0]["message"]["content"]
                usage = data.get("usage", {})
                p_tok = int(usage.get("prompt_tokens", 0))
                c_tok = int(usage.get("completion_tokens", 0))
                return content, p_tok, c_tok
        except Exception as e:
            raise RuntimeError(
                f"Model call failed ({url}, model={self.model}): {e}\n"
                "Make sure Ollama is running locally (ollama run llama3.2:3b) or set "
                "LLM_API_BASE, LLM_API_KEY, and LLM_MODEL environment variables."
            )


class HackathonAgentSystem:
    AREAS = ["venue", "schedule", "food", "judging", "budget"]

    def __init__(self, state_file: str = "plan_state.json"):
        self.state_file = state_file
        self.logger = TraceLogger()
        self.llm = LLMClient()
        self.messages = []
        self.plan = {area: "" for area in self.AREAS}
        self.rejected_messages = []

    def save_state(self):
        data = {
            "messages": self.messages,
            "plan": self.plan,
            "rejected_messages": self.rejected_messages,
            "trace": self.logger.calls,
            "total_tokens": self.logger.total_tokens()
        }
        with open(self.state_file, "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)
        with open("final_plan_and_trace.json", "w", encoding="utf-8") as f:
            json.dump(data, f, indent=2, ensure_ascii=False)

    def load_state(self) -> bool:
        if not os.path.exists(self.state_file):
            return False
        with open(self.state_file, "r", encoding="utf-8") as f:
            data = json.load(f)
        self.messages = data.get("messages", [])
        self.plan = data.get("plan", self.plan)
        self.rejected_messages = data.get("rejected_messages", [])
        self.logger.calls = data.get("trace", [])
        self.logger.call_counter = len(self.logger.calls)
        return True

    def read_messages(self, source: str) -> list[str]:
        if os.path.exists(source):
            with open(source, "r", encoding="utf-8") as f:
                raw = f.read().strip()
        else:
            raw = source
        return [line.strip() for line in raw.splitlines() if line.strip()]

    def call_area(self, area: str, system_prompt: str, user_prompt: str) -> str:
        output, p_tok, c_tok = self.llm.chat(system_prompt, user_prompt)
        self.logger.log(
            area=area,
            model=self.llm.model,
            input_text=user_prompt,
            output_text=output,
            prompt_tokens=p_tok,
            completion_tokens=c_tok
        )
        return output

    def run_initial_pack(self, source: str):
        self.messages = self.read_messages(source)
        pack_text = "\n".join(f"- {m}" for m in self.messages)

        reading_sys = (
            "You extract verified facts, numbers, dates, latest updates, and constraints "
            "from event messages in English and Roman Urdu. Ignore irrelevant chatter and "
            "supersede older prices/facts with newer dated messages. Be concise."
        )
        extracted_facts = self.call_area(
            "reading",
            reading_sys,
            f"Extract all valid facts and constraints from these messages:\n{pack_text}"
        )

        area_instructions = {
            "venue": "Select the best venue meeting capacity, power/generator, and budget constraints. Show exact figures.",
            "schedule": "Build a valid 4-hour event schedule respecting all breaks, start/end constraints, and judging time.",
            "food": "Calculate total headcount, select meal option using latest pricing, and compute total food cost.",
            "judging": "Verify judge count, max teams per judge, minutes per team, total judging time, and total judge honorarium cost.",
            "budget": "Itemize every committed and planned expense, verify total <= overall budget limit, and decide whether optional requests fit."
        }

        for area in self.AREAS:
            sys_p = f"You are the {area.upper()} planner for a physical hackathon. {area_instructions[area]} Rely ONLY on the provided facts."
            usr_p = f"Facts & Constraints:\n{extracted_facts}\n\nProduce the concise, exact {area} plan:"
            self.plan[area] = self.call_area(area, sys_p, usr_p)

        self.save_state()

    def process_new_message(self, source: str):
        new_lines = self.read_messages(source)
        new_msg = " ".join(new_lines)

        current_plan_summary = json.dumps(self.plan, indent=2)
        history_text = "\n".join(f"- {m}" for m in self.messages)

        val_sys = (
            "You validate a new incoming message against existing event facts and rules. "
            "Respond ONLY with valid JSON: "
            '{"valid": true/false, "reason": "clear explanation", "affected_areas": ["venue"|"schedule"|"food"|"judging"|"budget"]}. '
            "If the message violates capacity, headcount, budget, or rule constraints, set valid=false and affected_areas=[]."
        )
        val_prompt = (
            f"Existing Messages:\n{history_text}\n\n"
            f"Current Plan:\n{current_plan_summary}\n\n"
            f"New Message:\n{new_msg}"
        )
        raw_decision = self.call_area("reading", val_sys, val_prompt)

        decision = {"valid": True, "reason": "Processed update.", "affected_areas": ["budget"]}
        try:
            start = raw_decision.find("{")
            end = raw_decision.rfind("}") + 1
            if start != -1 and end > start:
                decision = json.loads(raw_decision[start:end])
        except Exception:
            pass

        if not decision.get("valid", True):
            reason = decision.get("reason", "Conflicts with existing event constraints.")
            self.rejected_messages.append({"message": new_msg, "reason": reason})
            print(f"\n[REJECTED MESSAGE] {new_msg}")
            print(f"[REASON] {reason}")
            self.save_state()
            return

        self.messages.append(new_msg)
        affected = [a for a in decision.get("affected_areas", []) if a in self.AREAS][:4]

        for area in affected:
            upd_sys = (
                f"Update ONLY the {area.upper()} section of the hackathon plan based on the new message. "
                "Keep all calculations exact and consistent with existing constraints."
            )
            upd_prompt = (
                f"Previous {area} plan:\n{self.plan[area]}\n\n"
                f"All Current Plan Context:\n{json.dumps(self.plan)}\n\n"
                f"New Valid Message:\n{new_msg}\n\n"
                f"Provide the updated {area} plan:"
            )
            self.plan[area] = self.call_area(area, upd_sys, upd_prompt)

        self.save_state()

    def render_ui(self):
        print("\n" + "=" * 72)
        print(" MIND TO MACHINE HACKATHON — EVENT PLANNER DASHBOARD")
        print("=" * 72)
        for area in self.AREAS:
            print(f"\n[{area.upper()}]")
            print(self.plan[area].strip())
        print("\n" + "-" * 72)
        print(" REJECTED MESSAGES")
        print("-" * 72)
        if not self.rejected_messages:
            print(" None.")
        else:
            for idx, item in enumerate(self.rejected_messages, 1):
                print(f" {idx}. Message: {item['message']}")
                print(f"    Reason : {item['reason']}")
        print("=" * 72)
        self.logger.print_trace()


def main():
    # If a file is passed as argument, use it. Otherwise use embedded default pack or file prompt.
    if len(sys.argv) >= 2:
        input_source = sys.argv[1]
    else:
        input_source = "messages.txt" if os.path.exists("messages.txt") else EMBEDDED_MESSAGES

    agent = HackathonAgentSystem()
    try:
        if not agent.load_state():
            agent.run_initial_pack(input_source)
        else:
            agent.process_new_message(input_source)
        agent.render_ui()
    except Exception as e:
        print(f"\n[ERROR] {e}")


if __name__ == "__main__":
    main()
