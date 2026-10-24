import 'dart:convert';
import 'dart:math' as math;
import 'package:flutter/material.dart';
import 'package:http/http.dart' as http;
import 'package:speech_to_text/speech_to_text.dart' as stt;
import 'package:audioplayers/audioplayers.dart';
import 'package:shared_preferences/shared_preferences.dart';

void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const JarvisApp());
}

enum JarvisState { idle, listening, processing, speaking }

class JarvisApp extends StatelessWidget {
  const JarvisApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'J.A.R.V.I.S.',
      debugShowCheckedModeBanner: false,
      theme: ThemeData.dark().copyWith(
        scaffoldBackgroundColor: const Color(0xFF030A16),
        primaryColor: const Color(0xFF00E5FF),
      ),
      home: const JarvisHomeScreen(),
    );
  }
}

class JarvisHomeScreen extends StatefulWidget {
  const JarvisHomeScreen({super.key});

  @override
  State<JarvisHomeScreen> createState() => _JarvisHomeScreenState();
}

class _JarvisHomeScreenState extends State<JarvisHomeScreen> with SingleTickerProviderStateMixin {
  JarvisState _state = JarvisState.idle;
  late stt.SpeechToText _speech;
  late AudioPlayer _audioPlayer;
  late AnimationController _animController;

  String _statusText = "Awaiting your command, sir.";
  String _liveTranscription = "";
  String _geminiApiKey = "";
  String _elevenLabsApiKey = "";
  String _voiceId = "pNInz6obpgDQGcFmaJgB"; // British Butler Voice

  final String _systemPrompt = 
    "You are J.A.R.V.I.S., Tony Stark's attentive, polite, and highly sophisticated British AI butler assistant. "
    "Always address the user as 'sir'. Speak in a calm, refined tone. "
    "CRITICAL: Responses must be brief and concise (1 to 3 sentences maximum) optimized for audio playback. "
    "Do NOT use markdown formatting, bullet points, asterisks, or emojis in spoken outputs.";

  @override
  void initState() {
    super.initState();
    _speech = stt.SpeechToText();
    _audioPlayer = AudioPlayer();
    _animController = AnimationController(
      vsync: this,
      duration: const Duration(seconds: 10),
    )..repeat();

    _loadPreferences();
  }

  Future<void> _loadPreferences() async {
    final prefs = await SharedPreferences.getInstance();
    setState(() {
      _geminiApiKey = prefs.getString('gemini_api_key') ?? '';
      _elevenLabsApiKey = prefs.getString('elevenlabs_api_key') ?? '';
      _voiceId = prefs.getString('elevenlabs_voice_id') ?? 'pNInz6obpgDQGcFmaJgB';
    });
  }

  void _toggleListening() async {
    if (_state == JarvisState.listening) {
      await _speech.stop();
      setState(() => _state = JarvisState.idle);
      if (_liveTranscription.isNotEmpty) {
        _processVoiceCommand(_liveTranscription);
      }
    } else {
      bool available = await _speech.initialize(
        onStatus: (val) {
          if (val == 'done') {
            setState(() => _state = JarvisState.processing);
            if (_liveTranscription.isNotEmpty) {
              _processVoiceCommand(_liveTranscription);
            }
          }
        },
        onError: (val) => print('STT Error: $val'),
      );
      if (available) {
        setState(() {
          _state = JarvisState.listening;
          _statusText = "Listening, sir...";
          _liveTranscription = "";
        });
        _speech.listen(onResult: (val) {
          setState(() {
            _liveTranscription = val.recognizedWords;
          });
        });
      }
    }
  }

  Future<void> _processVoiceCommand(String userInput) async {
    setState(() {
      _state = JarvisState.processing;
      _statusText = "Analyzing command: \"$userInput\"";
    });

    try {
      // 1. Brain: Gemini REST API generateContent
      final url = Uri.parse(
        'https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash:generateContent?key=$_geminiApiKey'
      );
      final geminiResponse = await http.post(
        url,
        headers: {'Content-Type': 'application/json'},
        body: jsonEncode({
          'system_instruction': {
            'parts': [{'text': _systemPrompt}]
          },
          'contents': [
            {
              'parts': [{'text': userInput}]
            }
          ]
        }),
      );

      String reply = "Indeed, sir.";
      if (geminiResponse.statusCode == 200) {
        final data = jsonDecode(geminiResponse.body);
        reply = data['candidates'][0]['content']['parts'][0]['text'] ?? reply;
      }

      setState(() {
        _statusText = reply;
        _state = JarvisState.speaking;
      });

      // 2. TTS: ElevenLabs REST API eleven_multilingual_v2
      await _speakWithElevenLabs(reply);

    } catch (e) {
      setState(() {
        _statusText = "Forgive me sir, a communication fault occurred.";
        _state = JarvisState.idle;
      });
    }
  }

  Future<void> _speakWithElevenLabs(String text) async {
    try {
      final ttsUrl = Uri.parse('https://api.elevenlabs.io/v1/text-to-speech/$_voiceId');
      final response = await http.post(
        ttsUrl,
        headers: {
          'xi-api-key': _elevenLabsApiKey,
          'Content-Type': 'application/json',
          'Accept': 'audio/mpeg',
        },
        body: jsonEncode({
          'text': text,
          'model_id': 'eleven_multilingual_v2',
          'voice_settings': {'stability': 0.65, 'similarity_boost': 0.85}
        }),
      );

      if (response.statusCode == 200) {
        await _audioPlayer.play(BytesSource(response.bodyBytes));
        _audioPlayer.onPlayerComplete.listen((_) {
          setState(() => _state = JarvisState.idle);
        });
      } else {
        setState(() => _state = JarvisState.idle);
      }
    } catch (_) {
      setState(() => _state = JarvisState.idle);
    }
  }

  Color _getStateColor() {
    switch (_state) {
      case JarvisState.listening:
        return const Color(0xFFFF1744); // Red
      case JarvisState.processing:
        return const Color(0xFFFFD700); // Yellow
      case JarvisState.speaking:
        return const Color(0xFF00E676); // Green
      case JarvisState.idle:
      default:
        return const Color(0xFF00E5FF); // Cyan
    }
  }

  @override
  Widget build(BuildContext context) {
    final stateColor = _getStateColor();

    return Scaffold(
      appBar: AppBar(
        backgroundColor: Colors.transparent,
        elevation: 0,
        title: const Text('J.A.R.V.I.S.', style: TextStyle(letterSpacing: 3, fontWeight: FontWeight.bold)),
        centerTitle: true,
      ),
      body: SafeArea(
        child: Column(
          children: [
            Expanded(
              child: Center(
                child: GestureDetector(
                  onTap: _toggleListening,
                  child: AnimatedBuilder(
                    animation: _animController,
                    builder: (context, child) {
                      return CustomPaint(
                        size: const Size(300, 300),
                        painter: ArcReactorPainter(
                          rotation: _animController.value * 2 * math.pi,
                          color: stateColor,
                          state: _state,
                        ),
                      );
                    },
                  ),
                ),
              ),
            ),
            Container(
              margin: const EdgeInsets.all(20),
              padding: const EdgeInsets.all(16),
              decoration: BoxDecoration(
                color: const Color(0xFF051224),
                borderRadius: BorderRadius.circular(16),
                border: Border.all(color: stateColor.withOpacity(0.4)),
              ),
              child: Column(
                crossAxisAlignment: CrossAxisAlignment.start,
                children: [
                  Text(_state.name.toUpperCase(), style: TextStyle(color: stateColor, fontSize: 10, letterSpacing: 2)),
                  const SizedBox(height: 6),
                  Text(_liveTranscription.isNotEmpty ? _liveTranscription : _statusText, style: const TextStyle(fontSize: 14)),
                ],
              ),
            ),
          ],
        ),
      ),
    );
  }
}

class ArcReactorPainter extends CustomPainter {
  final double rotation;
  final Color color;
  final JarvisState state;

  ArcReactorPainter({required this.rotation, required this.color, required this.state});

  @override
  void paint(Canvas canvas, Size size) {
    final center = Offset(size.width / 2, size.height / 2);
    final paint = Paint()
      ..color = color
      ..style = PaintingStyle.stroke
      ..strokeWidth = 2.0;

    // Outer ticks
    for (int i = 0; i < 36; i++) {
      double angle = rotation + (i * math.pi / 18);
      canvas.drawLine(
        Offset(center.dx + 120 * math.cos(angle), center.dy + 120 * math.sin(angle)),
        Offset(center.dx + 135 * math.cos(angle), center.dy + 135 * math.sin(angle)),
        paint,
      );
    }

    // Coils
    for (int i = 0; i < 10; i++) {
      double angle = -rotation * 0.5 + (i * 2 * math.pi / 10);
      canvas.drawCircle(
        Offset(center.dx + 80 * math.cos(angle), center.dy + 80 * math.sin(angle)),
        10,
        paint..style = PaintingStyle.fill,
      );
    }

    // Core
    canvas.drawCircle(center, 40, paint..color = color.withOpacity(0.7));
    canvas.drawCircle(center, 20, Paint()..color = Colors.white);
  }

  @override
  bool shouldRepaint(covariant CustomPainter oldDelegate) => true;
}
