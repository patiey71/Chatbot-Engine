# Chatbot-Engine
using System;
using System.Collections.Generic;
using System.Linq;

namespace CybersecurityChatbot
{
    // ─────────────────────────────────────────────
    //  Data class that holds user memory
    // ─────────────────────────────────────────────
    public class UserMemory
    {
        public string Name { get; set; } = string.Empty;
        public string FavouriteTopic { get; set; } = string.Empty;
        public List<string> Interests { get; set; } = new List<string>();

        public bool HasName => !string.IsNullOrEmpty(Name);
        public bool HasFavouriteTopic => !string.IsNullOrEmpty(FavouriteTopic);
    }

    // ─────────────────────────────────────────────
    //  Sentiment result
    // ─────────────────────────────────────────────
    public enum Sentiment { Neutral, Worried, Curious, Frustrated, Happy }

    // ─────────────────────────────────────────────
    //  Main chatbot engine (OOP, expandable)
    // ─────────────────────────────────────────────
    public class ChatbotEngine
    {
        // ── Memory ──────────────────────────────
        private readonly UserMemory _memory = new UserMemory();
        private string _lastTopic = string.Empty;
        private bool _awaitingName = false;
        private bool _awaitingTopic = false;

        // ── Random for varied responses ──────────
        private readonly Random _rng = new Random();

        // ── Keyword → responses dictionary ──────
        private readonly Dictionary<string, List<string>> _keywordResponses
            = new Dictionary<string, List<string>>(StringComparer.OrdinalIgnoreCase)
        {
            ["password"] = new List<string>
            {
                " Use a strong password with at least 12 characters mixing letters, numbers and symbols.",
                " Never reuse the same password on multiple sites — a password manager helps!",
                " Avoid using personal details like your name or birthday in passwords.",
                " Enable two-factor authentication (2FA) alongside a strong password for extra security."
            },
            ["phishing"] = new List<string>
            {
                " Be cautious of emails asking for personal information — scammers often disguise themselves as trusted organisations.",
                " Check the sender's email address carefully; phishing emails often use look-alike domains.",
                " Never click suspicious links in emails. Navigate to the website directly instead.",
                " Legitimate companies will never ask for your password via email."
            },
            ["scam"] = new List<string>
            {
                " Scammers can be very convincing. Always verify the identity of anyone requesting personal info.",
                " If an offer sounds too good to be true, it probably is — trust your instincts!",
                " Never send money or gift cards to people you have not met in person.",
                " Report scams to your local authorities or consumer protection agency."
            },
            ["privacy"] = new List<string>
            {
                " Review your social media privacy settings regularly to control who sees your data.",
                " Limit the personal information you share online — less is more when it comes to privacy.",
                " Use a VPN on public Wi-Fi networks to protect your browsing data.",
                " Read app permission requests carefully before granting access."
            },
            ["malware"] = new List<string>
            {
                " Keep your antivirus software up to date to guard against malware.",
                " Avoid downloading software from unofficial sources or unknown websites.",
                " Regularly scan your device for malware, especially after visiting new websites.",
                " Be wary of USB drives from unknown sources — they can contain malware."
            },
            ["ransomware"] = new List<string>
            {
                " Back up your important data regularly to an offline location to recover from ransomware.",
                " Keep your operating system and software patched to reduce ransomware vulnerability.",
                " Never pay the ransom — it does not guarantee file recovery and funds criminal activity."
            },
            ["firewall"] = new List<string>
            {
                " Ensure your device's firewall is enabled to block unauthorised network access.",
                " A hardware firewall on your router adds an extra layer of protection for all home devices.",
                " Review firewall logs occasionally to spot unusual connection attempts."
            },
            ["2fa"] = new List<string>
            {
                " Two-factor authentication (2FA) greatly reduces the risk of unauthorised account access.",
                " Use an authenticator app rather than SMS for stronger 2FA protection.",
                " Enable 2FA on all accounts that support it, especially email and banking."
            },
            ["vpn"] = new List<string>
            {
                " A VPN encrypts your internet traffic, protecting it on public Wi-Fi networks.",
                " Choose a reputable VPN provider — free VPNs may log and sell your data.",
                " A VPN hides your IP address but does not make you completely anonymous online."
            },
            ["social engineering"] = new List<string>
            {
                " Social engineering manipulates people rather than systems — always verify requests for sensitive info.",
                " Train yourself to pause and question urgent or unusual requests, even from 'known' contacts.",
                " Report any suspicious communication attempts to your IT department or organisation."
            }
        };

        // ── Sentiment keyword lists ──────────────
        private readonly List<string> _worriedWords   = new List<string> { "worried", "scared", "afraid", "anxious", "nervous", "concern", "fear" };
        private readonly List<string> _curiousWords   = new List<string> { "curious", "wonder", "interested", "how does", "what is", "tell me", "explain" };
        private readonly List<string> _frustratedWords = new List<string> { "frustrated", "annoyed", "angry", "hate", "useless", "not working", "confusing", "don't understand" };
        private readonly List<string> _happyWords     = new List<string> { "great", "awesome", "love", "thanks", "thank you", "helpful", "amazing", "good" };

        // ── Follow-up trigger words ──────────────
        private readonly List<string> _followUpTriggers = new List<string>
        {
            "another tip", "more", "tell me more", "explain more", "continue",
            "go on", "what else", "give me more", "another one", "next tip"
        };

        // ════════════════════════════════════════
        //  Public entry point
        // ════════════════════════════════════════
        public string GetResponse(string userInput)
        {
            if (string.IsNullOrWhiteSpace(userInput))
                return "Please type something so I can help you! ";

            string input = userInput.Trim();
            string lower = input.ToLower();

            // ── Handle name collection ──────────
            if (_awaitingName)
            {
                _awaitingName = false;
                _memory.Name = input;
                _awaitingTopic = true;
                return $"Nice to meet you, {_memory.Name}!  What is your favourite cybersecurity topic? " +
                       "(e.g. passwords, phishing, privacy, malware…)";
            }

            // ── Handle topic collection ─────────
            if (_awaitingTopic)
            {
                _awaitingTopic = false;
                _memory.FavouriteTopic = input;
                return $"Got it, {_memory.Name}! I'll keep in mind that you're interested in {_memory.FavouriteTopic}. " +
                       "Feel free to ask me anything about cybersecurity!";
            }

            // ── Greeting / intro ────────────────
            if (IsMatch(lower, new[] { "hello", "hi", "hey", "greetings", "good morning", "good afternoon", "good evening" }))
                return HandleGreeting();

            // ── Ask for name ────────────────────
            if (IsMatch(lower, new[] { "my name is", "i am", "i'm", "call me" }))
                return HandleNameIntroduction(input);

            // ── Help menu ───────────────────────
            if (IsMatch(lower, new[] { "help", "menu", "topics", "what can you do", "options" }))
                return GetHelpMenu();

            // ── Follow-up / continuation ─────────
            if (_followUpTriggers.Any(t => lower.Contains(t)))
                return HandleFollowUp();

            // ── Sentiment detection first ────────
            Sentiment sentiment = DetectSentiment(lower);
            string sentimentPrefix = GetSentimentPrefix(sentiment);

            // ── Keyword matching ────────────────
            foreach (var kvp in _keywordResponses)
            {
                if (lower.Contains(kvp.Key.ToLower()))
                {
                    _lastTopic = kvp.Key;

                    // Track interests for memory
                    if (!_memory.Interests.Contains(kvp.Key))
                        _memory.Interests.Add(kvp.Key);

                    string tip = kvp.Value[_rng.Next(kvp.Value.Count)];
                    return sentimentPrefix + tip + GetMemoryPersonalisation(kvp.Key);
                }
            }

            // ── Recall favourite topic ──────────
            if (IsMatch(lower, new[] { "my favourite", "my favorite", "my topic", "what do i like" }))
            {
                if (_memory.HasFavouriteTopic)
                    return $"You told me your favourite topic is {_memory.FavouriteTopic}! " +
                           $"Would you like a tip about it?";
                return "I don't know your favourite topic yet. What are you most interested in learning about?";
            }

            // ── Recall name ─────────────────────
            if (IsMatch(lower, new[] { "what is my name", "do you know my name", "my name" }))
            {
                if (_memory.HasName)
                    return $"Of course! You told me your name is {_memory.Name}. ";
                return "I don't know your name yet. Feel free to tell me — just say 'My name is ...'";
            }

            // ── Goodbye ─────────────────────────
            if (IsMatch(lower, new[] { "bye", "goodbye", "exit", "quit", "see you" }))
                return HandleGoodbye();

            // ── Default error handling ───────────
            return GetDefaultResponse();
        }

        // ════════════════════════════════════════
        //  Helper methods
        // ════════════════════════════════════════

        private string HandleGreeting()
        {
            if (_memory.HasName)
                return $"Hello again, {_memory.Name}!  Great to see you. How can I help you with cybersecurity today?";

            _awaitingName = true;
            return "Hello!  Welcome to CyberGuard Assistant. I'm here to help you stay safe online.\n\nWhat's your name?";
        }

        private string HandleNameIntroduction(string input)
        {
            // Extract name from phrases like "my name is John" or "I am Jane"
            string[] patterns = { "my name is ", "i am ", "i'm ", "call me " };
            string name = input;
            foreach (var p in patterns)
            {
                int idx = input.ToLower().IndexOf(p);
                if (idx >= 0)
                {
                    name = input.Substring(idx + p.Length).Trim();
                    // Remove trailing punctuation
                    name = name.TrimEnd('.', '!', ',');
                    break;
                }
            }
            _memory.Name = name;
            _awaitingTopic = true;
            return $"Nice to meet you, {_memory.Name}!  What is your favourite cybersecurity topic? " +
                   "(e.g. passwords, phishing, privacy, malware…)";
        }

        private string HandleFollowUp()
        {
            if (string.IsNullOrEmpty(_lastTopic))
                return "Sure! Which topic would you like to explore further? " +
                       "You can ask about passwords, phishing, scams, privacy, malware, and more!";

            if (_keywordResponses.ContainsKey(_lastTopic))
            {
                var tips = _keywordResponses[_lastTopic];
                string tip = tips[_rng.Next(tips.Count)];
                return $"Here's another tip about {_lastTopic}:\n\n{tip}";
            }

            return GetDefaultResponse();
        }

        private string HandleGoodbye()
        {
            string farewell = _memory.HasName
                ? $"Goodbye, {_memory.Name}!"
                : "Goodbye!";

            return $"{farewell}  Stay safe online. Remember to keep your software updated and use strong passwords!";
        }

        private string GetHelpMenu()
        {
            return " Here are the topics I can help you with:\n\n" +
                   "•  password  — Password safety tips\n" +
                   "•  phishing  — Recognising phishing attacks\n" +
                   "•  scam      — Avoiding online scams\n" +
                   "•  privacy   — Protecting your privacy\n" +
                   "•  malware   — Malware prevention\n" +
                   "•  ransomware — Ransomware defence\n" +
                   "•  firewall  — Firewall basics\n" +
                   "•  2fa       — Two-factor authentication\n" +
                   "•  vpn       — VPN usage\n" +
                   "•  social engineering — Social engineering\n\n" +
                   "Just type any topic or ask a question!";
        }

        private string GetDefaultResponse()
        {
            var defaults = new List<string>
            {
                "I'm not sure I understand. Can you try rephrasing? ",
                "Hmm, I didn't quite catch that. Could you ask about a specific cybersecurity topic?",
                "I'm not familiar with that. Try asking about passwords, phishing, or privacy!",
                "That's outside my knowledge. Type 'help' to see what I can assist with."
            };
            return defaults[_rng.Next(defaults.Count)];
        }

        // ── Sentiment detection ─────────────────
        public Sentiment DetectSentiment(string lower)
        {
            if (_worriedWords.Any(w => lower.Contains(w)))    return Sentiment.Worried;
            if (_frustratedWords.Any(w => lower.Contains(w))) return Sentiment.Frustrated;
            if (_happyWords.Any(w => lower.Contains(w)))      return Sentiment.Happy;
            if (_curiousWords.Any(w => lower.Contains(w)))    return Sentiment.Curious;
            return Sentiment.Neutral;
        }

        private string GetSentimentPrefix(Sentiment sentiment)
        {
            switch (sentiment)
            {
                case Sentiment.Worried:
                    return "It's completely understandable to feel that way. " +
                           "Cybersecurity can seem overwhelming, but you're already taking the right step by learning about it! \n\n";
                case Sentiment.Frustrated:
                    return "I understand this can be frustrating. Let me try to make this as clear as possible for you! \n\n";
                case Sentiment.Curious:
                    return "Great question! I love your curiosity. Here's what you need to know:\n\n";
                case Sentiment.Happy:
                    return "Glad you're feeling positive! Let me share a helpful tip:\n\n";
                default:
                    return string.Empty;
            }
        }

        // ── Memory personalisation ───────────────
        private string GetMemoryPersonalisation(string topic)
        {
            if (_memory.HasFavouriteTopic &&
                _memory.FavouriteTopic.IndexOf(topic, StringComparison.OrdinalIgnoreCase) >= 0)
            {
                return $"\n\n As someone interested in {_memory.FavouriteTopic}, you might want to review your account security settings too!";
            }

            if (_memory.HasName && _memory.Interests.Count > 1)
            {
                return $"\n\n {_memory.Name}, since you've been asking about {string.Join(" and ", _memory.Interests.TakeLast(2))}, " +
                       "consider doing a full security audit of your accounts!";
            }

            return string.Empty;
        }

        // ── Utility ─────────────────────────────
        private static bool IsMatch(string input, string[] keywords)
            => keywords.Any(k => input.Contains(k));

        // ── ASCII Art ────────────────────────────
        public static string GetAsciiArt()
        {
            return
                "   ██████╗██╗   ██╗██████╗ ███████╗██████╗  ██████╗ ██╗   ██╗ █████╗ ██████╗ ██████╗  \n" +
                "  ██╔════╝╚██╗ ██╔╝██╔══██╗██╔════╝██╔══██╗██╔════╝ ██║   ██║██╔══██╗██╔══██╗██╔══██╗ \n" +
                "  ██║      ╚████╔╝ ██████╔╝█████╗  ██████╔╝██║  ███╗██║   ██║███████║██████╔╝██║  ██║ \n" +
                "  ██║       ╚██╔╝  ██╔══██╗██╔══╝  ██╔══██╗██║   ██║██║   ██║██╔══██║██╔══██╗██║  ██║ \n" +
                "  ╚██████╗   ██║   ██████╔╝███████╗██║  ██║╚██████╔╝╚██████╔╝██║  ██║██║  ██║██████╔╝ \n" +
                "   ╚═════╝   ╚═╝   ╚═════╝ ╚══════╝╚═╝  ╚═╝ ╚═════╝  ╚═════╝ ╚═╝  ╚═╝╚═╝  ╚═╝╚═════╝  ";
        }
    }
}
