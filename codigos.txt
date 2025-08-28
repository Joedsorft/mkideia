Projeto MKIDEIA desenvolvedor - Mestre Kinho

Grupo 1
Flutter
Estrutura de pasta:

lib
   src
       api
            chat_agentes
               project_context_agent.dart
            api_service.dart
        controllers
            editor_controller.dart
            FileSystemController.dart
        models
            fs_node.dart
        pages
            center_panel.dart
            chat_page.dart
            home_page.dart
            left_panel.dart
            session_boot_page.dart
        providers
            model_provider.dart
            project_state.dart
            session_state.dart
        themes
            github_theme.dart
        utils
            file_service.dart
            language_config.dart
        widgets
            chat
                chat_action_bar.dart
                chat_header.dart
            action_bar.dart
            app_bar_widget.dart
            file_context_menu.dart
            file_system_tree.dart
            fs_exporter.dart
    main.dart
        
--------------------------------------------------------------------------------        
1. Codigo: api_service.dart
--------------------------------------------------------------------------------
import 'dart:async';
import 'dart:convert';
import 'dart:io';
import 'package:http/http.dart' as http;

class HybridApiService {
  final String baseUrl;
  bool useLocal;
  final http.Client _client = http.Client();

  String? currentProject = "default";
  String? currentSession;

  HybridApiService({
    required this.baseUrl,
    this.useLocal = false,
  });

  void dispose() {
    print('[DEBUG] Disposing HybridApiService');
    _client.close();
  }

  Future> getModels() async {
    print('[DEBUG] getModels called, useLocal=$useLocal');
    if (useLocal) {
      final result = await Process.run(
        'ollama',
        ['list'],
        stdoutEncoding: utf8,
        stderrEncoding: utf8,
      );
      if (result.exitCode == 0) {
        print('[DEBUG] Modelos locais obtidos');
        return result.stdout
            .toString()
            .split('\n')
            .skip(1)
            .where((l) => l.trim().isNotEmpty)
            .map((l) => l.split(' ').first)
            .toList();
      } else {
        throw Exception('Erro ao listar modelos locais: ${result.stderr}');
      }
    } else {
      final uri = Uri.parse('$baseUrl/models');
      final resp = await _client.get(uri).timeout(const Duration(seconds: 30));
      if (resp.statusCode == 200) {
        final j = jsonDecode(resp.body);
        if (j is Map && j['data'] is List) {
          print('[DEBUG] Modelos remotos obtidos');
          return (j['data'] as List).map((e) => e['id'].toString()).toList();
        }
      }
      throw Exception('Failed to load models: ${resp.statusCode} ${resp.body}');
    }
  }

  Future selectSession({required String projectId, required String sessionId}) async {
    print('[DEBUG] selectSession: project=$projectId session=$sessionId');
    currentProject = projectId;
    currentSession = sessionId;
  }

  Future _ensureSession({String projectId = "default"}) async {
    if (currentSession == null) {
      print('[DEBUG] _ensureSession criando sessionId');
      currentSession = DateTime.now().millisecondsSinceEpoch.toString();
      currentProject = projectId;
    }
  }

  Stream chatStreamWithHistory({
    required String model,
    required List> messages,
  }) async* {
    print('[DEBUG] chatStreamWithHistory called: model=$model messages=${messages.length}');
    if (useLocal) {
      final process = await Process.start('ollama', ['run', model], runInShell: true);
      for (var m in messages) {
        if (m['role'] == 'user') process.stdin.writeln(m['content']);
      }
      await process.stdin.flush();
      await process.stdin.close();

      await for (final data in process.stdout.transform(utf8.decoder)) {
        print('[DEBUG] Stream local delta: $data');
        yield data;
      }
    } else {
      await _ensureSession(projectId: currentProject ?? "default");

      final uri = Uri.parse('$baseUrl/chat/completions');
      final request = http.Request('POST', uri);
      request.headers['Content-Type'] = 'application/json';
      request.body = jsonEncode({
        "model": model,
        "messages": messages,
        "stream": true,
        "project_id": currentProject,
        "session_id": currentSession
      });

      final streamedResponse = await _client.send(request);
      if (streamedResponse.statusCode != 200) {
        final body = await streamedResponse.stream.bytesToString();
        throw Exception('Stream error: ${streamedResponse.statusCode} $body');
      }

      final lineStream = streamedResponse.stream.transform(utf8.decoder).transform(const LineSplitter());
      await for (final line in lineStream) {
        if (line.trim().isEmpty) continue;
        if (line.startsWith('data:')) {
          final payload = line.substring(5).trim();
          if (payload == '[DONE]') break;
          try {
            final parsed = jsonDecode(payload);
            if (parsed is Map && parsed.containsKey('delta')) {
              print('[DEBUG] Stream remoto delta: ${parsed['delta']}');
              yield parsed['delta'].toString();
            }
          } catch (_) {
            print('[DEBUG] Stream remoto nÃ£o JSON delta: $payload');
            yield payload;
          }
        }
      }
    }
  }

  /// Indexa um projeto jÃ¡ aberto (pasta raiz).
  /// Retorna um mapa com { "caminho/arquivo": "resumo/conteÃºdo" }
  Future> indexProject(String rootPath) async {
    print('[DEBUG] indexProject called: rootPath=$rootPath');
    final uri = Uri.parse('$baseUrl/index');
    final resp = await _client.post(
      uri,
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        "project_path": rootPath,
        "project_id": currentProject ?? "default",
      }),
    ).timeout(const Duration(minutes: 2));

    if (resp.statusCode == 200) {
      final data = jsonDecode(resp.body);
      if (data is Map && data['files'] is Map) {
        final files = Map.from(data['files']);
        print('[DEBUG] Projeto indexado: ${files.length} arquivos');
        return files;
      }
    }
    throw Exception('Erro ao indexar projeto: ${resp.statusCode} ${resp.body}');
  }

  /// Faz uma consulta sobre o projeto jÃ¡ indexado.
  Future queryProject(String query) async {
    print('[DEBUG] queryProject called: query=$query');
    final uri = Uri.parse('$baseUrl/query');
    final resp = await _client.post(
      uri,
      headers: {'Content-Type': 'application/json'},
      body: jsonEncode({
        "query": query,
        "project_id": currentProject ?? "default",
      }),
    ).timeout(const Duration(seconds: 60));

    if (resp.statusCode == 200) {
      final data = jsonDecode(resp.body);
      if (data is Map && data.containsKey('answer')) {
        return data['answer'].toString();
      }
    }
    throw Exception('Erro ao consultar projeto: ${resp.statusCode} ${resp.body}');
  }
}

--------------------------------------------------------------------------------            
2. Codigo: session_boot_page.dart
--------------------------------------------------------------------------------
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../api/api_service.dart';
import '../providers/session_state.dart';
import 'chat_page.dart';

class SessionBootPage extends StatefulWidget {
  const SessionBootPage({super.key});
  @override
  State createState() => _SessionBootPageState();
}

class _SessionBootPageState extends State {
  bool _loading = true;
  String? _error;

  @override
  void initState() {
    super.initState();
    print('[DEBUG] initState SessionBootPage');
    _boot();
  }

  Future _boot() async {
    print('[DEBUG] _boot called');
    try {
      final state = context.read();
      final api = context.read();

      final nowId = DateTime.now().millisecondsSinceEpoch.toString();
      print('[DEBUG] Criando nova sessÃ£o: $nowId');
      state.setSession(proj: state.projectId, sess: nowId);
      api.currentProject = state.projectId;
      api.currentSession = nowId;

    } catch (e) {
      print('[DEBUG] Erro no boot: $e');
      _error = '$e';
    } finally {
      if (mounted) {
        print('[DEBUG] Boot finalizado');
        setState(() => _loading = false);
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    print('[DEBUG] Build SessionBootPage');
    if (_loading) return const Scaffold(body: Center(child: CircularProgressIndicator()));
    if (_error != null) return Scaffold(body: Center(child: Text('Falha ao iniciar sessÃ£o: $_error')));
    return ChatPage(getProjectRootPath: () {  },);
  }
}

--------------------------------------------------------------------------------
3. Codigo: project_state.dart
--------------------------------------------------------------------------------
// providers/project_state.dart
import 'package:flutter/material.dart';

class ProjectState extends ChangeNotifier {
  String? rootPath;
  Map indexedFiles = {};

  void setRoot(String path) {
    rootPath = path;
    indexedFiles.clear();
    print('[DEBUG][ProjectState] RootPath setado: $rootPath');
    notifyListeners();
  }

  void setIndex(Map files) {
    indexedFiles = files;
    print('[DEBUG][ProjectState] IndexedFiles atualizado: ${indexedFiles.keys.toList()}');
    notifyListeners();
  }


  bool get isIndexed => indexedFiles.isNotEmpty;
}

--------------------------------------------------------------------------------
4. Codigo: main.dart
--------------------------------------------------------------------------------
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import 'src/api/api_service.dart';
import 'src/pages/home_page.dart';
import 'src/providers/model_provider.dart';
import 'src/providers/session_state.dart';

void main() {
  WidgetsFlutterBinding.ensureInitialized();
  runApp(const MKIdeApp());
}

class MKIdeApp extends StatelessWidget {
  const MKIdeApp({super.key});

  @override
  Widget build(BuildContext context) {
    final darkTheme = ThemeData.dark().copyWith(
      scaffoldBackgroundColor: const Color(0xFF111214),
      canvasColor: const Color(0xFF111214),
      cardColor: const Color(0xFF151617),
      dividerColor: const Color(0xFF2A2C2D),
      appBarTheme: const AppBarTheme(
        elevation: 0,
        backgroundColor: Color(0xFF111214),
        foregroundColor: Colors.white,
      ),
      inputDecorationTheme: const InputDecorationTheme(
        filled: true,
        fillColor: Color(0xFF1C212B),
        border: OutlineInputBorder(
          borderSide: BorderSide(color: Color(0xFF2A2C2D)),
        ),
        enabledBorder: OutlineInputBorder(
          borderSide: BorderSide(color: Color(0xFF2A2C2D)),
        ),
        focusedBorder: OutlineInputBorder(
          borderSide: BorderSide(color: Colors.white30),
        ),
      ),
      dialogTheme: const DialogThemeData(
        backgroundColor: Color(0xFF171A22),
      ),
      colorScheme: ThemeData.dark().colorScheme.copyWith(
        primary: Colors.blueGrey.shade200,
        secondary: Colors.blueGrey.shade300,
      ),
      textTheme: ThemeData.dark().textTheme,
    );

    return MultiProvider(
      providers: [
        ChangeNotifierProvider(
          create: (_) => SessionState(),
        ),
        Provider(
          create: (_) => HybridApiService(
            baseUrl: 'http://127.0.0.1:5000/v1',
            useLocal: false,
          ),
        ),
        ChangeNotifierProvider(
          create: (_) => ModelProvider(),
        ),
      ],
      child: MaterialApp(
        debugShowCheckedModeBanner: false,
        theme: darkTheme,
        home: const HomePage(), // mantÃ©m o layout SplitView original
      ),
    );
  }
}

--------------------------------------------------------------------------------
5. Codigo: home_page.dart
--------------------------------------------------------------------------------
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter_code_editor/flutter_code_editor.dart';
import 'package:path/path.dart' as p;
import 'package:split_view/split_view.dart';
import 'package:file_picker/file_picker.dart';
import '../controllers/FileSystemController.dart';
import '../models/fs_node.dart';
import '../utils/language_config.dart';
import '../controllers/editor_controller.dart';
import 'left_panel.dart';
import 'center_panel.dart';
import 'chat_page.dart';

class HomePage extends StatefulWidget {
  const HomePage({super.key});

  @override
  State createState() => _HomePageState();
}

class _HomePageState extends State {
  FsNode? _rootNode;
  String? _selectedPath;
  FileSystemEntity? selectedFile;

  EditorController? _editorController;
  bool _isEditing = false;

  final SplitViewController horizontalSplitController =
  SplitViewController(weights: [0.25, 0.5, 0.25]);

  FileSystemController? _fsController;

  @override
  void dispose() {
    _editorController?.codeController.dispose();
    horizontalSplitController.dispose();
    super.dispose();
  }

  // ====== Abrir pasta ======
  Future _openFolder() async {
    String? selectedDirectory = await FilePicker.platform.getDirectoryPath();
    if (selectedDirectory != null) {
      _fsController = FileSystemController(selectedDirectory);
      final tree = await _buildFsNode(Directory(selectedDirectory));
      setState(() {
        _rootNode = tree;
        _selectedPath = null;
        selectedFile = null;
        _editorController = null;
        _isEditing = false;
      });
    }
  }

  Future _buildFsNode(Directory dir) async {
    final children = [];
    for (final f in dir.listSync()) {
      if (f is Directory) {
        final child = await _buildFsNode(f);
        children.add(child);
      } else if (f is File) {
        children.add(FsNode(
          name: f.uri.pathSegments.last,
          path: f.path,
          isDir: false,
        ));
      }
    }

    children.sort((a, b) {
      if (a.isDir && !b.isDir) return -1;
      if (!a.isDir && b.isDir) return 1;
      return a.name.toLowerCase().compareTo(b.name.toLowerCase());
    });

    return FsNode(
      name: dir.path.split(Platform.pathSeparator).last.isNotEmpty
          ? dir.path.split(Platform.pathSeparator).last
          : dir.path,
      path: dir.path,
      isDir: true,
      children: children,
    );
  }

  // ====== Abrir arquivo ======
  void _openFileByPath(String path) {
    final f = File(path);
    if (!f.existsSync()) return;

    try {
      final content = f.readAsStringSync();
      final language = getLanguageForPath(path);

      _editorController?.codeController.dispose();

      setState(() {
        selectedFile = f;
        _selectedPath = path;
        _editorController = EditorController(
          CodeController(text: content, language: language, patternMap: {}),
        );
        _isEditing = false;
      });
      _editorController?.commitEditingSession();
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('Erro ao abrir arquivo: $e')),
      );
      setState(() {
        selectedFile = null;
        _selectedPath = null;
        _editorController =
            EditorController(CodeController(text: '', language: plainText));
        _isEditing = false;
      });
      _editorController?.commitEditingSession();
    }
  }

  // ====== Salvar ======
  void _saveFile() {
    if (selectedFile != null && _editorController != null) {
      try {
        File(selectedFile!.path).writeAsStringSync(_editorController!.text);
        _editorController!.commitEditingSession();
        setState(() => _isEditing = false);
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Arquivo salvo com sucesso!')),
        );
      } catch (e) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Erro ao salvar arquivo: $e')),
        );
      }
    }
  }

  // ====== Salvar como ======
  Future _saveFileAs() async {
    if (_editorController == null) return;

    String? savePath = await FilePicker.platform.saveFile(
      dialogTitle: 'Salvar arquivo como',
      fileName: selectedFile?.uri.pathSegments.last ?? 'novo_arquivo.txt',
    );

    if (savePath != null) {
      try {
        File(savePath).writeAsStringSync(_editorController!.text);
        setState(() {
          _selectedPath = savePath;
          selectedFile = File(savePath);
          _isEditing = false;
        });
        _editorController!.commitEditingSession();
        ScaffoldMessenger.of(context).showSnackBar(
          const SnackBar(content: Text('Arquivo salvo com sucesso!')),
        );
      } catch (e) {
        ScaffoldMessenger.of(context).showSnackBar(
          SnackBar(content: Text('Erro ao salvar arquivo: $e')),
        );
      }
    }
  }

  // ====== Undo/Redo ======
  void _undo() {
    _editorController?.undo();
    setState(() => _isEditing = true);
  }

  void _redo() {
    _editorController?.redo();
    setState(() => _isEditing = true);
  }

  // ====== Context Menu ======
  void _showContextMenu(BuildContext context, Offset position, FsNode node) {
    if (_fsController == null) return;

    final overlay = Overlay.of(context).context.findRenderObject() as RenderBox?;
    final relativePosition = overlay?.globalToLocal(position) ?? Offset.zero;

    showMenu(
      context: context,
      position: RelativeRect.fromLTRB(
        relativePosition.dx,
        relativePosition.dy,
        relativePosition.dx,
        relativePosition.dy,
      ),
      items: [
        PopupMenuItem(
          child: const Text("Novo Arquivo"),
          onTap: () async {
            await _fsController!.createFile(p.join(node.path, "novo_arquivo"));
            _openFolder();
          },
        ),
        PopupMenuItem(
          child: const Text("Nova Pasta"),
          onTap: () async {
            await _fsController!.createFolder(p.join(node.path, "Nova_Pasta"));
            _openFolder();
          },
        ),
        PopupMenuItem(
          child: const Text("Renomear"),
          onTap: () async {
            // Pode abrir diÃ¡logo de renome
          },
        ),
        PopupMenuItem(
          child: const Text("Deletar"),
          onTap: () async {
            await _fsController!.delete(node.path);
            _openFolder();
          },
        ),
      ],
    );
  }

  PreferredSizeWidget _buildTopAppBar() {
    return AppBar(
      title: const Text('MKIDEIA - FlowEngine'),
      elevation: 0,
      backgroundColor: const Color(0xFF111214),
    );
  }

  PreferredSizeWidget _buildMenuBar() {
    return AppBar(
      backgroundColor: const Color(0xFF0F1115),
      toolbarHeight: 40,
      titleSpacing: 0,
      title: Row(
        children: [
          IconButton(
            icon: const Icon(Icons.folder_open, color: Colors.white),
            onPressed: _openFolder,
            tooltip: 'Open Folder',
          ),
          IconButton(
            icon: const Icon(Icons.save, color: Colors.white),
            onPressed: _saveFile,
            tooltip: 'Save',
          ),
          IconButton(
            icon: const Icon(Icons.save_as, color: Colors.white),
            onPressed: _saveFileAs,
            tooltip: 'Save As',
          ),
          IconButton(
            icon: const Icon(Icons.undo, color: Colors.white),
            onPressed: _undo,
            tooltip: 'Undo',
          ),
          IconButton(
            icon: const Icon(Icons.redo, color: Colors.white),
            onPressed: _redo,
            tooltip: 'Redo',
          ),
        ],
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: _buildTopAppBar(),
      body: Column(
        children: [
          _buildMenuBar(),
          Expanded(
            child: SplitView(
              controller: horizontalSplitController,
              gripSize: 8,
              gripColor: const Color(0xFF2A2C2D),
              viewMode: SplitViewMode.Horizontal,
              children: [
                LeftPanel(
                  rootNode: _rootNode,
                  selectedPath: _selectedPath,
                  onFileTap: _openFileByPath,
                  onOpenFolder: _openFolder,
                  onRightClick: (offset, node) {
                    _showContextMenu(context, offset, node);
                  },
                ),
                CenterPanel(
                  codeController: _editorController?.codeController ??
                      CodeController(text: '', language: plainText),
                  isEditing: _isEditing,
                  onCodeChange: (text) {
                    _editorController?.onUserChange();
                    setState(() => _isEditing = true);
                  },
                ),
                ChatPage(
                  getProjectRootPath: () {
                    print('[DEBUG][HomePage] getProjectRootPath chamado');
                    if (_rootNode != null) {
                      print('[DEBUG][HomePage] rootNode.path = ${_rootNode!.path}');
                      return _rootNode!.path;
                    }
                    print('[DEBUG][HomePage] Nenhum projeto aberto, retornando null');
                    return null;
                  },
                ),


              ],
            ),
          ),
        ],
      ),
    );
  }
}

--------------------------------------------------------------------------------
6. Codigo: chat_page.dart
--------------------------------------------------------------------------------
import 'dart:async';
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:provider/provider.dart';

import '../api/api_service.dart';
import '../api/chat_agentes/project_context_agent.dart';
import '../providers/session_state.dart';
import '../widgets/chat/chat_action_bar.dart';
import '../widgets/chat/chat_header.dart';

class ChatPage extends StatefulWidget {
  final String? Function() getProjectRootPath; // novo parÃ¢metro

  const ChatPage({
    super.key,
    required this.getProjectRootPath, // obrigatÃ³rio
  });

  @override
  State createState() => _ChatPageState();
}

class _ChatPageState extends State {
  final TextEditingController _chatController = TextEditingController();
  final ScrollController _chatScrollController = ScrollController();
  StreamSubscription? _streamSub;

  bool _useLocal = true; // controla se estÃ¡ em modo Local ou Servidor
  bool _projectIndexed = false; // se o botÃ£o de indexaÃ§Ã£o estÃ¡ ativo
  Map _projectIndex = {}; // conteÃºdo indexado do projeto
  ProjectContextAgent? _contextAgent;
  String? _currentProjectId;

  @override
  void initState() {
    super.initState();
    _initProjectContext();
  }

  Future _initProjectContext() async {
    final rootPath = widget.getProjectRootPath?.call();
    if (rootPath != null) {
      _currentProjectId = rootPath.split(Platform.pathSeparator).last;
      _contextAgent = ProjectContextAgent(_currentProjectId!);
      final loaded = await _contextAgent!.loadContext();
      _projectIndex = loaded;
      setState(() {
        _projectIndexed = _projectIndex.isNotEmpty;
      });
      print('[ChatPage] Projeto carregado com contexto: ${_projectIndex.keys}');
    } else {
      print('[ChatPage] Nenhum projeto aberto ao inicializar contexto');
    }
  }

  @override
  void dispose() {
    _streamSub?.cancel();
    _chatController.dispose();
    _chatScrollController.dispose();
    super.dispose();
  }

  void _sendMessage() {
    final prompt = _chatController.text.trim();
    if (prompt.isEmpty) return;

    final state = context.read();
    final api = context.read();

    if (state.model.isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Selecione um modelo')),
      );
      return;
    }

    state.addUser(prompt);
    state.startAssistant();
    _chatController.clear();
    _scrollChatToBottom();

    final msgs = state.messages
        .map((m) => {'role': m.role, 'content': m.content})
        .toList();

    // ðŸ”¹ Injeta contexto atualizado se o projeto estiver indexado
    if (_projectIndexed && _projectIndex.isNotEmpty) {
      final contextText = _projectIndex.entries
          .map((e) => '${e.key}: ${e.value}')
          .join('\n\n');
      msgs.insert(0, {'role': 'system', 'content': 'Contexto do projeto:\n$contextText'});
      print('[ChatPage] Contexto injetado: ${_projectIndex.keys}');
    }

    _streamSub?.cancel();
    _streamSub = api
        .chatStreamWithHistory(model: state.model, messages: msgs)
        .listen((delta) {
      state.appendAssistant(delta);
      _scrollChatToBottom();
    }, onError: (e) {
      state.appendAssistant('\n[Erro: $e]');
      state.endStream();
    }, onDone: () {
      state.endStream();
    });
  }

  void _scrollChatToBottom() {
    WidgetsBinding.instance.addPostFrameCallback((_) {
      if (_chatScrollController.hasClients) {
        _chatScrollController.animateTo(
          _chatScrollController.position.maxScrollExtent,
          duration: const Duration(milliseconds: 150),
          curve: Curves.easeOut,
        );
      }
    });
  }

  void _editMessage(SessionState state, int index) {
    final msg = state.messages[index];
    _chatController.text = msg.content;
    state.removeAt(index);
  }

  void _removeMessage(SessionState state, int index) {
    state.removeAt(index);
  }

  Future _toggleProjectIndexation() async {
    final rootPath = widget.getProjectRootPath?.call();
    if (rootPath == null) {
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Abra um projeto antes de indexar')),
      );
      return;
    }

    setState(() {
      _projectIndexed = !_projectIndexed;
    });

    if (_projectIndexed) {
      _currentProjectId = rootPath.split(Platform.pathSeparator).last;
      _contextAgent ??= ProjectContextAgent(_currentProjectId!);
      _projectIndex = await _contextAgent!.loadContext();
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Projeto indexado e contexto ativo')),
      );
      print('[ChatPage] Contexto atualizado: ${_projectIndex.keys}');
    } else {
      await _contextAgent?.clearContext();
      _projectIndex.clear();
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Contexto do projeto desativado')),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    final state = context.watch();

    return Column(
      children: [
        // ðŸ”¹ CabeÃ§alho do chat
        ChatHeader(
          useLocal: _useLocal,
          streaming: state.streaming,
          onOpenChat: () {
            ScaffoldMessenger.of(context).showSnackBar(
              const SnackBar(content: Text('Chat aberto!')),
            );
          },
          onStopStream: () {
            _streamSub?.cancel();
            state.endStream();
          },
          onConnectionChange: (v) {
            setState(() => _useLocal = v);
          },
        ),

        // ðŸ”¹ Corpo do chat (mensagens)
        Expanded(
          child: ListView.builder(
            controller: _chatScrollController,
            padding: const EdgeInsets.all(12),
            itemCount: state.messages.length,
            itemBuilder: (_, i) {
              final m = state.messages[i];
              final isUser = m.role == 'user';
              return Container(
                padding: const EdgeInsets.symmetric(vertical: 8, horizontal: 12),
                alignment: isUser ? Alignment.centerRight : Alignment.centerLeft,
                child: ConstrainedBox(
                  constraints: const BoxConstraints(maxWidth: 800),
                  child: DecoratedBox(
                    decoration: BoxDecoration(
                      color: isUser ? const Color(0xFF0F1115) : const Color(0xFF2A1B3D),
                      borderRadius: BorderRadius.circular(8),
                    ),
                    child: Padding(
                      padding: const EdgeInsets.all(12),
                      child: Column(
                        crossAxisAlignment:
                        isUser ? CrossAxisAlignment.end : CrossAxisAlignment.start,
                        children: [
                          SelectableText(
                            m.content,
                            showCursor: true,
                            cursorWidth: 2,
                            cursorColor: Colors.purple,
                            cursorRadius: const Radius.circular(2),
                            style: const TextStyle(color: Colors.white),
                          ),
                          if (isUser) ...[
                            const SizedBox(height: 8),
                            Row(
                              mainAxisSize: MainAxisSize.min,
                              children: [
                                IconButton(
                                  icon: const Icon(Icons.copy, size: 18, color: Colors.white),
                                  tooltip: 'Copiar',
                                  onPressed: () {
                                    Clipboard.setData(ClipboardData(text: m.content));
                                    ScaffoldMessenger.of(context).showSnackBar(
                                      const SnackBar(content: Text('Copiado para Ã¡rea de transferÃªncia')),
                                    );
                                  },
                                ),
                                if (isUser) ...[
                                  IconButton(
                                    icon: const Icon(Icons.edit, size: 18, color: Colors.white),
                                    tooltip: 'Editar',
                                    onPressed: state.streaming ? null : () => _editMessage(state, i),
                                  ),
                                  IconButton(
                                    icon: const Icon(Icons.delete, size: 18, color: Colors.white),
                                    tooltip: 'Remover',
                                    onPressed: state.streaming ? null : () => _removeMessage(state, i),
                                  ),
                                ],
                              ],
                            ),
                          ],
                        ],
                      ),
                    ),
                  ),
                ),
              );
            },
          ),
        ),

        // ðŸ”¹ Action bar com os botÃµes extras
        ChatActionBar(
          useLocal: _useLocal,
          onClearMessages: () => state.clearMessages(),
          getLastAssistantText: () => state.messages
              .lastWhere((m) => m.role == 'assistant', orElse: () => ChatMessage("assistant", ""))
              .content,
          onConnectionChange: (v) {
            setState(() => _useLocal = v);
          },
          getProjectRootPath: widget.getProjectRootPath,
          key: const ValueKey('chatActionBar'),
        ),

        // ðŸ”¹ BotÃ£o verde de indexaÃ§Ã£o do projeto
        Padding(
          padding: const EdgeInsets.symmetric(vertical: 6),
          child: ElevatedButton.icon(
            icon: const Icon(Icons.folder_open),
            label: Text(_projectIndexed ? 'Projeto Indexado' : 'Indexar Projeto'),
            style: ElevatedButton.styleFrom(
              backgroundColor: _projectIndexed ? Colors.green : Colors.grey[800],
            ),
            onPressed: _toggleProjectIndexation,
          ),
        ),

        // ðŸ”¹ Campo de input
        const Divider(height: 1, color: Color(0xFF222426)),
        SafeArea(
          top: false,
          child: Padding(
            padding: const EdgeInsets.symmetric(horizontal: 8.0, vertical: 6),
            child: Row(
              children: [
                Expanded(
                  child: TextField(
                    controller: _chatController,
                    minLines: 1,
                    maxLines: 5,
                    decoration: const InputDecoration(
                      hintText: 'Digite sua mensagem...',
                    ),
                    enabled: !state.streaming,
                    onSubmitted: (_) => _sendMessage(),
                  ),
                ),
                IconButton(
                  icon: state.streaming
                      ? const SizedBox(
                    width: 20,
                    height: 20,
                    child: CircularProgressIndicator(strokeWidth: 2),
                  )
                      : const Icon(Icons.send),
                  onPressed: state.streaming ? null : _sendMessage,
                ),
              ],
            ),
          ),
        ),
      ],
    );
  }
}

--------------------------------------------------------------------------------
7. Codigo: left_panel.dart
--------------------------------------------------------------------------------
import 'package:flutter/material.dart';
import '../models/fs_node.dart';

class LeftPanel extends StatefulWidget {
  final FsNode? rootNode;
  final String? selectedPath;
  final Function(String) onFileTap;
  final Function() onOpenFolder;
  final Function(Offset position, FsNode node)? onRightClick;

  const LeftPanel({
    super.key,
    required this.rootNode,
    required this.selectedPath,
    required this.onFileTap,
    required this.onOpenFolder,
    this.onRightClick,
  });

  /// âœ… Expor caminho raiz do projeto aberto
  String? get projectRootPath => rootNode?.path;

  @override
  State createState() => _LeftPanelState();
}

class _LeftPanelState extends State {
  String? _hoveredPath;
  late final ScrollController _scrollController;

  @override
  void initState() {
    super.initState();
    _scrollController = ScrollController();
  }

  @override
  void dispose() {
    _scrollController.dispose();
    super.dispose();
  }

  Widget _buildTree(FsNode node, {int depth = 0}) {
    final paddingLeft = depth == 0 ? 12.0 : 12.0 + depth * 16.0;

    if (node.isDir) {
      return Padding(
        padding: EdgeInsets.only(left: paddingLeft),
        child: Theme(
          data: Theme.of(context).copyWith(dividerColor: Colors.transparent),
          child: GestureDetector(
            onSecondaryTapDown: (details) {
              if (widget.onRightClick != null) {
                widget.onRightClick!(details.globalPosition, node);
              }
            },
            child: ExpansionTile(
              initiallyExpanded: depth == 0,
              leading: const Icon(Icons.folder, color: Colors.amber),
              title: Text(
                node.name,
                style: const TextStyle(
                  color: Colors.white,
                  fontWeight: FontWeight.bold,
                ),
              ),
              children: node.children
                  .map((c) => _buildTree(c, depth: depth + 1))
                  .toList(),
            ),
          ),
        ),
      );
    } else {
      final isSelected = node.path == widget.selectedPath;
      final isHovered = node.path == _hoveredPath;

      return Padding(
        padding: EdgeInsets.only(left: paddingLeft),
        child: MouseRegion(
          onEnter: (_) => setState(() => _hoveredPath = node.path),
          onExit: (_) => setState(() => _hoveredPath = null),
          child: GestureDetector(
            onTap: () => widget.onFileTap(node.path),
            onSecondaryTapDown: (details) {
              if (widget.onRightClick != null) {
                widget.onRightClick!(details.globalPosition, node);
              }
            },
            child: Container(
              decoration: BoxDecoration(
                color: isSelected
                    ? const Color(0xFF2A2C2D)
                    : isHovered
                    ? const Color(0xFF1F2123)
                    : Colors.transparent,
                borderRadius: BorderRadius.circular(6),
              ),
              child: ListTile(
                dense: true,
                leading: const Icon(Icons.insert_drive_file, color: Colors.grey),
                title: Text(
                  node.name,
                  style: TextStyle(
                    color: isSelected || isHovered ? Colors.white : Colors.white70,
                    fontWeight: isSelected ? FontWeight.bold : FontWeight.normal,
                  ),
                ),
                selected: isSelected,
              ),
            ),
          ),
        ),
      );
    }
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      decoration: const BoxDecoration(
        color: Color(0xFF0F1112),
        border: Border(right: BorderSide(color: Color(0xFF222426))),
      ),
      child: Column(
        children: [
          // CabeÃ§alho
          Container(
            width: double.infinity,
            padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 10),
            decoration: const BoxDecoration(
              border: Border(bottom: BorderSide(color: Color(0xFF222426))),
            ),
            child: Row(
              mainAxisAlignment: MainAxisAlignment.spaceBetween,
              children: [
                Expanded(
                  child: Text(
                    widget.rootNode?.path ?? 'Nenhum projeto aberto',
                    maxLines: 1,
                    overflow: TextOverflow.ellipsis,
                    style: const TextStyle(color: Colors.white70, fontSize: 12),
                  ),
                ),
                IconButton(
                  icon: const Icon(Icons.folder_open, color: Colors.white70),
                  tooltip: 'Open Folder',
                  onPressed: widget.onOpenFolder,
                ),
              ],
            ),
          ),
          // Lista de arquivos/pastas
          Expanded(
            child: widget.rootNode == null
                ? const Center(
              child: Text(
                'Use File > Open Folder',
                style: TextStyle(color: Colors.white70),
              ),
            )
                : Scrollbar(
              controller: _scrollController,
              thumbVisibility: true,
              child: ListView(
                controller: _scrollController,
                padding: const EdgeInsets.symmetric(horizontal: 6, vertical: 8),
                children: [_buildTree(widget.rootNode!)],
              ),
            ),
          ),
          const Divider(height: 1, color: Color(0xFF222426)),
          // RodapÃ© com ajuda e configuraÃ§Ãµes
          Padding(
            padding: const EdgeInsets.all(12),
            child: Column(
              children: const [
                Row(
                  children: [
                    Icon(Icons.help_outline, color: Colors.white70, size: 18),
                    SizedBox(width: 8),
                    Text('Ajuda', style: TextStyle(color: Colors.white70)),
                  ],
                ),
                SizedBox(height: 8),
                Row(
                  children: [
                    Icon(Icons.settings, color: Colors.white70, size: 18),
                    SizedBox(width: 8),
                    Text('ConfiguraÃ§Ãµes',
                        style: TextStyle(color: Colors.white70)),
                  ],
                ),
              ],
            ),
          ),
        ],
      ),
    );
  }
}

--------------------------------------------------------------------------------
8. Codigo: chat_action_bar.dart
--------------------------------------------------------------------------------
import 'dart:io';
import 'package:file_picker/file_picker.dart';
import 'package:path/path.dart' as p;
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';

import '../../api/api_service.dart';
import '../../api/chat_agentes/project_context_agent.dart';
import '../../controllers/editor_controller.dart';
class ChatActionBar extends StatefulWidget {
  final VoidCallback onClearMessages;
  final String? Function() getLastAssistantText;
  final ValueChanged onConnectionChange;
  final bool useLocal;

  /// âœ… Novo: funÃ§Ã£o para pegar caminho do projeto aberto
  final String? Function() getProjectRootPath;

  const ChatActionBar({
    super.key,
    required this.onClearMessages,
    required this.getLastAssistantText,
    required this.onConnectionChange,
    required this.useLocal,
    required this.getProjectRootPath, // âœ… obrigatÃ³rio agora
  });

  @override
  State createState() => _ChatActionBarState();
}

class _ChatActionBarState extends State {
  final Map> _undoStacks = {};
  final Map> _redoStacks = {};

  bool _projectIndexed = false;
  Map _projectIndex = {};
  ProjectContextAgent? _contextAgent;
  String? _currentProjectId;

  Future _toggleProjectIndex() async {
    final dir = widget.getProjectRootPath();
    print('[DEBUG][ChatActionBar] _toggleProjectIndex chamado');
    print('[DEBUG][ChatActionBar] getProjectRootPath() retornou: $dir');
    print('[DEBUG][ChatActionBar] _projectIndexed = $_projectIndexed');

    if (dir == null || dir.isEmpty) {
      print('[DEBUG][ChatActionBar] Nenhum projeto aberto. Abortando indexaÃ§Ã£o.');
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Abra um projeto antes de indexar')),
      );
      return;
    }

    if (!_projectIndexed) {
      print('[DEBUG][ChatActionBar] Indexando projeto...');
      _projectIndex.clear();
      await _indexDirectory(dir);

      _currentProjectId = dir.split(Platform.pathSeparator).last;
      print('[DEBUG][ChatActionBar] _currentProjectId = $_currentProjectId');

      _contextAgent = ProjectContextAgent(_currentProjectId!);
      await _contextAgent?.saveContext(_projectIndex);

      setState(() => _projectIndexed = true);
      print('[DEBUG][ChatActionBar] Projeto indexado com sucesso.');
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('Projeto indexado')),
      );
    } else {
      print('[DEBUG][ChatActionBar] Desativando indexaÃ§Ã£o...');
      _projectIndex.clear();
      await _contextAgent?.clearContext();
      _contextAgent = null;
      _currentProjectId = null;
      setState(() => _projectIndexed = false);
      ScaffoldMessenger.of(context).showSnackBar(
        const SnackBar(content: Text('IndexaÃ§Ã£o desativada')),
      );
    }
  }


  Future _indexDirectory(String dir) async {
    final entityList = Directory(dir).listSync(recursive: true);
    for (var entity in entityList) {
      if (entity is File) {
        try {
          final content = entity.readAsStringSync();
          _projectIndex[entity.path] =
              content.substring(0, content.length.clamp(0, 500));
        } catch (_) {}
      }
    }
  }


  // âœ… Novo helper para injetar contexto no prompt da IA
  Future>> injectContext(List> messages) async {
    if (_contextAgent == null) return messages;
    return await _contextAgent!.injectContextIntoMessages(messages);
  }

  // ----------------- Helpers de UI -----------------
  Future _confirm(String title, String message) async {
    final res = await showDialog(
      context: context,
      builder: (c) => AlertDialog(
        title: Text(title),
        content: Text(message),
        actions: [
          TextButton(onPressed: () => Navigator.of(c).pop(false), child: const Text('Cancelar')),
          ElevatedButton(onPressed: () => Navigator.of(c).pop(true), child: const Text('Confirmar')),
        ],
      ),
    );
    return res == true;
  }

  Future _promptForText(String title, {String hint = '', String initial = ''}) async {
    final ctrl = TextEditingController(text: initial);
    final res = await showDialog(
      context: context,
      builder: (c) => AlertDialog(
        title: Text(title),
        content: TextField(
          controller: ctrl,
          decoration: InputDecoration(hintText: hint),
          autofocus: true,
        ),
        actions: [
          TextButton(onPressed: () => Navigator.of(c).pop(null), child: const Text('Cancelar')),
          ElevatedButton(onPressed: () => Navigator.of(c).pop(ctrl.text.trim()), child: const Text('OK')),
        ],
      ),
    );
    return res;
  }

  void _pushUndo(String path, String previousContent) {
    final stack = _undoStacks.putIfAbsent(path, () => []);
    stack.add(previousContent);
    _redoStacks.remove(path);
  }

  void _pushRedo(String path, String content) {
    final stack = _redoStacks.putIfAbsent(path, () => []);
    stack.add(content);
  }

  Future _notifyEditorIfProviderExists(String path, String content) async {
    try {
      final ec = Provider.of(context, listen: false);
      ec.updateText(content);
    } catch (_) {}
  }

  // ----------------- OperaÃ§Ãµes de arquivo -----------------
  Future _undoFile(String path) async {
    try {
      final uStack = _undoStacks[path];
      if (uStack == null || uStack.isEmpty) {
        ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Nada para desfazer')));
        return;
      }
      final current = File(path).existsSync() ? File(path).readAsStringSync() : '';
      final last = uStack.removeLast();
      _pushRedo(path, current);
      File(path).writeAsStringSync(last);
      await _notifyEditorIfProviderExists(path, last);
      ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Undo aplicado')));
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Erro no undo: $e')));
    }
  }

  Future _redoFile(String path) async {
    try {
      final rStack = _redoStacks[path];
      if (rStack == null || rStack.isEmpty) {
        ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Nada para refazer')));
        return;
      }
      final current = File(path).existsSync() ? File(path).readAsStringSync() : '';
      final last = rStack.removeLast();
      _pushUndo(path, current);
      File(path).writeAsStringSync(last);
      await _notifyEditorIfProviderExists(path, last);
      ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Redo aplicado')));
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Erro no redo: $e')));
    }
  }

  Future _createFile() async {
    String? dir = await FilePicker.platform.getDirectoryPath(dialogTitle: 'Escolha pasta onde criar o arquivo');
    if (dir == null) return;

    final filename = await _promptForText('Nome do arquivo', hint: 'ex: novo_arquivo.txt', initial: 'novo_arquivo.txt');
    if (filename == null || filename.isEmpty) return;

    final fullPath = p.join(dir, filename);
    if (File(fullPath).existsSync()) {
      final overwrite = await _confirm('Arquivo existe', 'Arquivo jÃ¡ existe. Deseja sobrescrever?');
      if (!overwrite) return;
    }

    String initialContent = '';
    final candidate = widget.getLastAssistantText.call();
    if (candidate != null && candidate.trim().isNotEmpty) {
      initialContent = candidate;
    }

    try {
      _pushUndo(fullPath, File(fullPath).existsSync() ? File(fullPath).readAsStringSync() : '');
      File(fullPath).writeAsStringSync(initialContent);
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Arquivo criado: ${p.basename(fullPath)}')));
      await _notifyEditorIfProviderExists(fullPath, initialContent);
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Erro criando arquivo: $e')));
    }
  }

  Future _createFolder() async {
    final dir = await FilePicker.platform.getDirectoryPath(dialogTitle: 'Escolha pasta onde criar a nova pasta');
    if (dir == null) return;

    final folderName = await _promptForText('Nome da pasta', hint: 'ex: nova_pasta', initial: 'nova_pasta');
    if (folderName == null || folderName.isEmpty) return;

    final fullPath = p.join(dir, folderName);
    if (Directory(fullPath).existsSync()) {
      ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Pasta jÃ¡ existe')));
      return;
    }

    try {
      Directory(fullPath).createSync(recursive: true);
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Pasta criada: $fullPath')));
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Erro criando pasta: $e')));
    }
  }

  Future _deletePath() async {
    final result = await FilePicker.platform.pickFiles(allowMultiple: false);
    if (result == null || result.files.single.path == null) return;

    final path = result.files.single.path!;
    final confirm = await _confirm('Confirmar exclusÃ£o', 'Deseja excluir permanentemente: ${p.basename(path)} ?');
    if (!confirm) return;

    try {
      if (File(path).existsSync()) {
        _pushUndo(path, File(path).readAsStringSync());
        File(path).deleteSync();
      } else if (Directory(path).existsSync()) {
        _pushUndo(path, '<>');
        Directory(path).deleteSync(recursive: true);
      }
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('ExcluÃ­do: ${p.basename(path)}')));
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Erro ao excluir: $e')));
    }
  }

  Future _renamePath() async {
    final result = await FilePicker.platform.pickFiles(allowMultiple: false);
    if (result == null || result.files.single.path == null) return;

    final oldPath = result.files.single.path!;
    final newName = await _promptForText('Novo nome', hint: 'novo_nome.ext', initial: p.basename(oldPath));
    if (newName == null || newName.isEmpty) return;

    final newPath = p.join(p.dirname(oldPath), newName);
    try {
      if (File(oldPath).existsSync()) {
        _pushUndo(oldPath, File(oldPath).readAsStringSync());
        File(oldPath).renameSync(newPath);
      } else if (Directory(oldPath).existsSync()) {
        _pushUndo(oldPath, '<>');
        Directory(oldPath).renameSync(newPath);
      }
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Renomeado para: $newPath')));
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Erro renomeando: $e')));
    }
  }

  Future _applyAIPatchToFile() async {
    final lastAssistant = widget.getLastAssistantText.call();
    if (lastAssistant == null || lastAssistant.trim().isEmpty) {
      ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Sem mensagens do assistant para aplicar')));
      return;
    }

    final result = await FilePicker.platform.pickFiles(allowMultiple: false);
    if (result == null || result.files.single.path == null) return;

    final targetPath = result.files.single.path!;
    final confirm = await _confirm(
      'Aplicar alteraÃ§Ã£o',
      'Deseja aplicar o conteÃºdo do assistant ao arquivo ${p.basename(targetPath)} ?',
    );
    if (!confirm) return;

    try {
      final previous = File(targetPath).existsSync() ? File(targetPath).readAsStringSync() : '';
      _pushUndo(targetPath, previous);
      File(targetPath).writeAsStringSync(lastAssistant);
      await _notifyEditorIfProviderExists(targetPath, lastAssistant);
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text('ConteÃºdo aplicado a ${p.basename(targetPath)}')),
      );
    } catch (e) {
      ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Erro aplicando patch: $e')));
    }
  }

  // ----------------- UI -----------------
  Widget _roundBtn(IconData icon, String tooltip, Future Function()? onTap,
      {bool active = false}) {
    return Tooltip(
      message: tooltip,
      child: InkWell(
        onTap: onTap == null ? null : () async => await onTap(),
        borderRadius: BorderRadius.circular(28),
        child: CircleAvatar(
          radius: 22,
          backgroundColor: active ? Colors.green : const Color(0xFF1A1B1C),
          child: Icon(icon, color: Colors.white, size: 20),
        ),
      ),
    );
  }

  @override
  Widget build(BuildContext context) {
    final api = Provider.of(context, listen: false);

    return Container(
      padding: const EdgeInsets.symmetric(horizontal: 10, vertical: 8),
      decoration: const BoxDecoration(
        color: Color(0xFF141516),
        border: Border(
          top: BorderSide(color: Color(0xFF222426)),
          bottom: BorderSide(color: Color(0xFF222426)),
        ),
      ),
      child: Wrap(
        spacing: 10,
        runSpacing: 8,
        children: [
          _roundBtn(Icons.chat_bubble_outline, 'Nova conversa', () async {
            widget.onClearMessages();
            ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Nova conversa iniciada')));
          }),
          _roundBtn(Icons.push_pin_outlined, 'Fixar resumo', () async {
            ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Resumo fixado (placeholder)')));
          }),
          _roundBtn(Icons.folder_open, 'Indexar projeto', _toggleProjectIndex, active: _projectIndexed),
          _roundBtn(Icons.smart_toy_outlined, 'Executar agente', () async {
            ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Agente executando (placeholder)')));
          }),
          _roundBtn(Icons.language, 'Ativar Internet', () async {
            try {
              api.useLocal = false;
              widget.onConnectionChange(false);
              ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Internet ativada (modo HTTP)')));
            } catch (e) {
              ScaffoldMessenger.of(context).showSnackBar(SnackBar(content: Text('Erro ao ativar internet: $e')));
            }
          }),
          _roundBtn(Icons.refresh, 'Reindexar', () async {
            ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Reindexando (placeholder)')));
          }),
          _roundBtn(Icons.image_outlined, 'Importar imagem', () async {
            ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Importar imagem (placeholder)')));
          }),
          _roundBtn(Icons.auto_fix_high, 'Wizards', () async {
            ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Wizards (placeholder)')));
          }),
          _roundBtn(Icons.note_add, 'Criar arquivo', _createFile),
          _roundBtn(Icons.create_new_folder, 'Criar pasta', _createFolder),
          _roundBtn(Icons.delete_outline, 'Excluir arquivo/pasta', _deletePath),
          _roundBtn(Icons.drive_file_rename_outline, 'Renomear', _renamePath),
          _roundBtn(Icons.code, 'Aplicar conteÃºdo (IA) a arquivo', _applyAIPatchToFile),
          _roundBtn(Icons.undo, 'Undo (arquivo)', () async {
            final result = await FilePicker.platform.pickFiles(allowMultiple: false);
            if (result == null || result.files.single.path == null) return;
            await _undoFile(result.files.single.path!);
          }),
          _roundBtn(Icons.redo, 'Redo (arquivo)', () async {
            final result = await FilePicker.platform.pickFiles(allowMultiple: false);
            if (result == null || result.files.single.path == null) return;
            await _redoFile(result.files.single.path!);
          }),
        ],
      ),
    );
  }
}

---------------------------------------------------------------------------------------
9. Codigo: chat_header.dart
---------------------------------------------------------------------------------------
import 'package:flutter/material.dart';
import 'package:provider/provider.dart';
import '../../api/api_service.dart';
import '../../providers/model_provider.dart';

class ChatHeader extends StatelessWidget {
  final bool useLocal;
  final bool streaming;
  final VoidCallback onOpenChat;
  final VoidCallback onStopStream;
  final ValueChanged onConnectionChange;

  const ChatHeader({
    super.key,
    required this.useLocal,
    required this.streaming,
    required this.onOpenChat,
    required this.onStopStream,
    required this.onConnectionChange,
  });

  @override
  Widget build(BuildContext context) {
    final modelProv = Provider.of(context);
    final api = Provider.of(context, listen: false);

    return Column(
      children: [
        // Linha 1: ConexÃ£o (Local/Servidor)
        Container(
          alignment: Alignment.centerLeft,
          color: const Color(0xFF0F1115),
          height: 36,
          padding: const EdgeInsets.symmetric(horizontal: 12),
          child: Row(
            children: [
              const Text('ConexÃ£o:', style: TextStyle(color: Colors.white70)),
              const SizedBox(width: 12),
              DropdownButton(
                value: useLocal,
                dropdownColor: const Color(0xFF1C212B),
                items: const [
                  DropdownMenuItem(value: true, child: Text("Local (Terminal)")),
                  DropdownMenuItem(value: false, child: Text("Servidor HTTP")),
                ],
                onChanged: (v) async {
                  if (v == null) return;
                  try {
                    // atualiza o serviÃ§o e avisa o chat_page via callback
                    api.useLocal = v;
                    onConnectionChange(v);
                    ScaffoldMessenger.of(context).showSnackBar(
                      SnackBar(content: Text(v ? 'ConexÃ£o: Local' : 'ConexÃ£o: Servidor HTTP')),
                    );
                  } catch (e) {
                    ScaffoldMessenger.of(context).showSnackBar(
                      SnackBar(content: Text('Erro ao alternar conexÃ£o: $e')),
                    );
                  }
                },
              ),
              const SizedBox(width: 8),
              // opcional: botÃ£o para forÃ§ar reload dos modelos
              IconButton(
                tooltip: 'Recarregar modelos',
                icon: const Icon(Icons.refresh, color: Colors.white70),
                onPressed: () async {
                  try {
                    final models = await api.getModels();
                    modelProv.setModels(models);
                    ScaffoldMessenger.of(context).showSnackBar(
                      const SnackBar(content: Text('Modelos recarregados')),
                    );
                  } catch (e) {
                    ScaffoldMessenger.of(context).showSnackBar(
                      SnackBar(content: Text('Erro carregando modelos: $e')),
                    );
                  }
                },
              ),
            ],
          ),
        ),

        // Linha 2: Seletor de modelo + Abrir Chat + Parar
        Container(
          alignment: Alignment.centerLeft,
          color: const Color(0xFF0F1115),
          height: 54,
          padding: const EdgeInsets.symmetric(horizontal: 12),
          child: Row(
            children: [
              const Text('Modelo:', style: TextStyle(color: Colors.white70)),
              const SizedBox(width: 12),
              Expanded(
                child: DropdownButton(
                  value: modelProv.selectedModel.isEmpty ? null : modelProv.selectedModel,
                  dropdownColor: const Color(0xFF1C212B),
                  items: modelProv.models
                      .map((m) => DropdownMenuItem(value: m, child: Text(m)))
                      .toList(),
                  onChanged: (v) {
                    if (v != null) modelProv.selectModel(v);
                  },
                  isExpanded: true,
                  hint: const Text('Selecione um modelo'),
                ),
              ),
              const SizedBox(width: 8),
              ElevatedButton(
                onPressed: () {
                  if (modelProv.selectedModel.isEmpty) {
                    ScaffoldMessenger.of(context).showSnackBar(
                      const SnackBar(content: Text('Selecione um modelo primeiro')),
                    );
                  } else {
                    onOpenChat();
                  }
                },
                child: const Text('Abrir Chat'),
              ),
              if (streaming)
                Padding(
                  padding: const EdgeInsets.only(left: 8.0),
                  child: TextButton.icon(
                    icon: const Icon(Icons.stop, color: Colors.red),
                    label: const Text('Parar'),
                    onPressed: onStopStream,
                  ),
                ),
            ],
          ),
        ),
      ],
    );
  }
}


--------------------------------------------------------------------------------
10. Codigo: project_context_agent.dart
--------------------------------------------------------------------------------
import 'dart:convert';
import 'dart:io';
import 'package:path/path.dart' as p;

class ProjectContextAgent {
  final String projectId;

  ProjectContextAgent(this.projectId);

  /// Caminho fixo de persistÃªncia (~/.mkide/projects/{projectId}/context.json)
  String get _contextFilePath {
    // Caminho fixo para indexes
    final base = p.join('/home/mk/mkide_ia/data/indexes', projectId);
    Directory(base).createSync(recursive: true);
    return p.join(base, 'context.json');
  }


  /// Salva contexto (mapa de arquivos com snippet inicial ou resumo)
  Future saveContext(Map index) async {
    final f = File(_contextFilePath);
    await f.writeAsString(jsonEncode(index), flush: true);
    print('[DEBUG][ProjectContextAgent] Contexto salvo em $_contextFilePath');
    print('[DEBUG][ProjectContextAgent] Arquivos indexados: ${index.keys.toList()}');
  }

  /// Carrega contexto existente
  Future> loadContext() async {
    final f = File(_contextFilePath);
    if (!f.existsSync()) return {};
    try {
      final raw = await f.readAsString();
      final decoded = Map.from(jsonDecode(raw));
      print('[ProjectContextAgent] Arquivos carregados do context.json: ${decoded.keys}');
      return decoded;
    } catch (e) {
      print('[ProjectContextAgent] Erro ao carregar contexto: $e');
      return {};
    }
  }


  /// Limpa contexto
  Future clearContext() async {
    final f = File(_contextFilePath);
    if (f.existsSync()) {
      await f.delete();
      print('[DEBUG][ProjectContextAgent] Contexto limpo em $_contextFilePath');
    } else {
      print('[DEBUG][ProjectContextAgent] Nenhum contexto para limpar em $_contextFilePath');
    }
  }

  /// Injeta contexto no prompt
  Future>> injectContextIntoMessages(
      List> messages) async {
    final ctx = await loadContext();
    if (ctx.isEmpty) return messages;

    final summary = ctx.entries
        .map((e) => 'ðŸ“„ ${p.basename(e.key)}\n${e.value}')
        .join('\n\n');

    return [
      {
        "role": "system",
        "content":
        "O seguinte contexto do projeto foi indexado e deve ser considerado:\n$summary"
      },
      ...messages
    ];
  }
}


--------------------------------------------------------------------------------
11. Codigo: fs_node.dart
--------------------------------------------------------------------------------
// Modelo de nÃ³ da Ã¡rvore de arquivos/pastas
import 'dart:io';

class FsNode {
  final String name;
  final String path;
  final bool isDir;
  final List children;

  FsNode({
    required this.name,
    required this.path,
    required this.isDir,
    this.children = const [],
  });

  FsNode copyWith({List? children}) =>
      FsNode(name: name, path: path, isDir: isDir, children: children ?? this.children);

  /// Cria a Ã¡rvore a partir de um Directory
  factory FsNode.fromDirectory(Directory dir) {
    final children = [];
    for (var entity in dir.listSync(recursive: false)) {
      if (entity is Directory) {
        children.add(FsNode.fromDirectory(entity));
      } else if (entity is File) {
        children.add(FsNode(name: entity.uri.pathSegments.last, path: entity.path, isDir: false));
      }
    }
    return FsNode(
      name: dir.uri.pathSegments.isNotEmpty ? dir.uri.pathSegments.last : dir.path,
      path: dir.path,
      isDir: true,
      children: children,
    );
  }
}


--------------------------------------------------------------------------------
12. Codigo: editor_controller.dart
--------------------------------------------------------------------------------
import 'package:flutter_code_editor/flutter_code_editor.dart';

//EditorController â†’ Ã© focado no que estÃ¡ aberto no editor central (tabs, texto carregado, alteraÃ§Ãµes, salvar, etc). Ele gerencia o conteÃºdo que o usuÃ¡rio estÃ¡ editando.

/// Controla o estado do editor (texto, undo/redo) e padroniza
/// o ponto de integraÃ§Ã£o com a IA e a UI.
class EditorController {
  final CodeController codeController;

  final List _undoStack = [];
  final List _redoStack = [];

  /// Texto "commitado" (Ãºltimo estado estÃ¡vel apÃ³s salvar/abrir/undo/redo).
  String _committedText;

  /// Indica se jÃ¡ abrimos uma sessÃ£o de ediÃ§Ã£o desde o Ãºltimo commit.
  bool _editingSessionOpen = false;

  EditorController(this.codeController) : _committedText = codeController.text;

  /// Chamado quando a UI (digitaÃ§Ã£o do usuÃ¡rio) altera o texto.
  /// Na primeira mudanÃ§a, empilha o snapshot anterior no undo.
  void onUserChange() {
    if (!_editingSessionOpen) {
      _undoStack.add(_committedText);
      _redoStack.clear();
      _editingSessionOpen = true;
    }
    // NÃ£o mexemos em codeController.text aqui, o CodeField jÃ¡ atualizou.
  }

  /// Atualiza o texto de forma programÃ¡tica (ex.: IA aplicando patch).
  /// Empilha o estado atual no undo e reseta sessÃ£o.
  void updateText(String newText) {
    _undoStack.add(codeController.text);
    _redoStack.clear();
    codeController.text = newText;
    _committedText = newText;
    _editingSessionOpen = false;
  }

  /// Confirma o estado atual como base (apÃ³s salvar/abrir/undo/redo).
  void commitEditingSession() {
    _committedText = codeController.text;
    _editingSessionOpen = false;
  }

  void undo() {
    // Fecha sessÃ£o aberta para nÃ£o perder consistÃªncia.
    if (_editingSessionOpen) {
      commitEditingSession();
    }
    if (_undoStack.isNotEmpty) {
      final prev = _undoStack.removeLast();
      _redoStack.add(codeController.text);
      codeController.text = prev;
      _committedText = codeController.text;
      _editingSessionOpen = false;
    }
  }

  void redo() {
    if (_editingSessionOpen) {
      commitEditingSession();
    }
    if (_redoStack.isNotEmpty) {
      final next = _redoStack.removeLast();
      _undoStack.add(codeController.text);
      codeController.text = next;
      _committedText = codeController.text;
      _editingSessionOpen = false;
    }
  }

  String get text => codeController.text;
}


--------------------------------------------------------------------------------
13. Codigo: FileSystemController.dart
--------------------------------------------------------------------------------
import 'package:flutter_code_editor/flutter_code_editor.dart';

//EditorController â†’ Ã© focado no que estÃ¡ aberto no editor central (tabs, texto carregado, alteraÃ§Ãµes, salvar, etc). Ele gerencia o conteÃºdo que o usuÃ¡rio estÃ¡ editando.

/// Controla o estado do editor (texto, undo/redo) e padroniza
/// o ponto de integraÃ§Ã£o com a IA e a UI.
class EditorController {
  final CodeController codeController;

  final List _undoStack = [];
  final List _redoStack = [];

  /// Texto "commitado" (Ãºltimo estado estÃ¡vel apÃ³s salvar/abrir/undo/redo).
  String _committedText;

  /// Indica se jÃ¡ abrimos uma sessÃ£o de ediÃ§Ã£o desde o Ãºltimo commit.
  bool _editingSessionOpen = false;

  EditorController(this.codeController) : _committedText = codeController.text;

  /// Chamado quando a UI (digitaÃ§Ã£o do usuÃ¡rio) altera o texto.
  /// Na primeira mudanÃ§a, empilha o snapshot anterior no undo.
  void onUserChange() {
    if (!_editingSessionOpen) {
      _undoStack.add(_committedText);
      _redoStack.clear();
      _editingSessionOpen = true;
    }
    // NÃ£o mexemos em codeController.text aqui, o CodeField jÃ¡ atualizou.
  }

  /// Atualiza o texto de forma programÃ¡tica (ex.: IA aplicando patch).
  /// Empilha o estado atual no undo e reseta sessÃ£o.
  void updateText(String newText) {
    _undoStack.add(codeController.text);
    _redoStack.clear();
    codeController.text = newText;
    _committedText = newText;
    _editingSessionOpen = false;
  }

  /// Confirma o estado atual como base (apÃ³s salvar/abrir/undo/redo).
  void commitEditingSession() {
    _committedText = codeController.text;
    _editingSessionOpen = false;
  }

  void undo() {
    // Fecha sessÃ£o aberta para nÃ£o perder consistÃªncia.
    if (_editingSessionOpen) {
      commitEditingSession();
    }
    if (_undoStack.isNotEmpty) {
      final prev = _undoStack.removeLast();
      _redoStack.add(codeController.text);
      codeController.text = prev;
      _committedText = codeController.text;
      _editingSessionOpen = false;
    }
  }

  void redo() {
    if (_editingSessionOpen) {
      commitEditingSession();
    }
    if (_redoStack.isNotEmpty) {
      final next = _redoStack.removeLast();
      _undoStack.add(codeController.text);
      codeController.text = next;
      _committedText = codeController.text;
      _editingSessionOpen = false;
    }
  }

  String get text => codeController.text;
}


--------------------------------------------------------------------------------
14. CÃ³digo: model_provider.dart
--------------------------------------------------------------------------------
//codigo atual
import 'package:flutter/material.dart';

class ModelProvider extends ChangeNotifier {
  List models = [];
  String selectedModel = '';

  void setModels(List m) {
    models = m;
    if (m.isNotEmpty && selectedModel.isEmpty) {
      selectedModel = m[0];
    }
    notifyListeners();
  }

  void selectModel(String model) {
    selectedModel = model;
    notifyListeners();
  }
}


--------------------------------------------------------------------------------
15. Codigo: file_service.dart
--------------------------------------------------------------------------------
// lib/src/utils/file_service.dart

import 'dart:io';
import 'package:path/path.dart' as p;

/// ServiÃ§o simples de operaÃ§Ãµes em arquivos/pastas com backup bÃ¡sico.
/// - backups sÃ£o guardados em /.mkideia_backups//
class FileService {
  FileService();

  /// Cria um arquivo (e diretÃ³rio pai se necessÃ¡rio) com conteÃºdo inicial.
  /// Retorna true se criado/existente com sucesso.
  Future createFile(String path, {String content = ''}) async {
    final f = File(path);
    final dir = f.parent;
    if (!await dir.exists()) {
      await dir.create(recursive: true);
    }
    if (!await f.exists()) {
      await f.writeAsString(content);
      return true;
    } else {
      // jÃ¡ existe: nÃ£o sobrescreve
      return false;
    }
  }

  /// Cria pasta (recursiva). Retorna true se criada/existente.
  Future createFolder(String path) async {
    final d = Directory(path);
    if (!await d.exists()) {
      await d.create(recursive: true);
      return true;
    }
    return false;
  }

  /// Renomeia/Move arquivo ou pasta.
  Future rename(String oldPath, String newPath) async {
    final e = FileSystemEntity.typeSync(oldPath);
    if (e == FileSystemEntityType.directory) {
      final dir = Directory(oldPath);
      await dir.rename(newPath);
    } else {
      final f = File(oldPath);
      await f.rename(newPath);
    }
  }

  /// Apaga arquivo ou pasta (recursivo para pastas).
  Future delete(String path) async {
    final t = FileSystemEntity.typeSync(path);
    if (t == FileSystemEntityType.directory) {
      final d = Directory(path);
      if (await d.exists()) await d.delete(recursive: true);
    } else {
      final f = File(path);
      if (await f.exists()) await f.delete();
    }
  }

  /// Escreve conteÃºdo em arquivo. Faz backup antes (se arquivo existir).
  /// Retorna old content (antes da escrita) para diffs/undo.
  Future writeFile(String path, String content) async {
    final f = File(path);
    String? old;
    if (await f.exists()) {
      old = await f.readAsString();
      await _backupFile(path, old);
    } else {
      final dir = f.parent;
      if (!await dir.exists()) await dir.create(recursive: true);
      old = null;
    }
    await f.writeAsString(content);
    return old;
  }

  /// Copia recursivamente uma pasta (use para "Salvar projeto como").
  Future copyDirectory(String srcPath, String destPath) async {
    final src = Directory(srcPath);
    if (!await src.exists()) throw Exception('Origem nÃ£o encontrada: $srcPath');
    final dest = Directory(destPath);
    if (!await dest.exists()) await dest.create(recursive: true);

    await for (final entity in src.list(recursive: true, followLinks: false)) {
      final relative = p.relative(entity.path, from: srcPath);
      final newPath = p.join(destPath, relative);
      if (entity is Directory) {
        final d = Directory(newPath);
        if (!await d.exists()) await d.create(recursive: true);
      } else if (entity is File) {
        final f = File(newPath);
        if (!await f.parent.exists()) await f.parent.create(recursive: true);
        await entity.copy(newPath);
      }
    }
  }

  /// Faz backup de um arquivo para .mkideia_backups//
  Future _backupFile(String path, String content) async {
    try {
      final file = File(path);
      final projectRoot = _findProjectRoot(path) ?? file.parent.path;
      final now = DateTime.now().toIso8601String().replaceAll(':', '-');
      final backupDir = Directory(p.join(projectRoot, '.mkideia_backups', now));
      if (!await backupDir.exists()) await backupDir.create(recursive: true);
      final rel = p.relative(path, from: projectRoot);
      final backupPath = p.join(backupDir.path, rel);
      final backupFile = File(backupPath);
      if (!await backupFile.parent.exists()) await backupFile.parent.create(recursive: true);
      await backupFile.writeAsString(content);
    } catch (_) {
      // nÃ£o estourar exceÃ§Ã£o de backup
    }
  }

  /// Tenta localizar a raiz do projeto olhando para a pasta que contÃ©m 'pubspec.yaml'
  /// sobe atÃ© encontrar ou retorna null.
  String? _findProjectRoot(String startPath) {
    var dir = Directory(startPath);
    if (!dir.existsSync()) dir = dir.parent;
    while (dir.path != dir.parent.path) {
      final pub = File(p.join(dir.path, 'pubspec.yaml'));
      if (pub.existsSync()) return dir.path;
      dir = dir.parent;
    }
    return null;
  }

  /// Gera um diff simples (linhas removidas/adicionadas) entre old/new
  Map> simpleLineDiff(String? oldStr, String newStr) {
    final oldLines = (oldStr ?? '').split('\n');
    final newLines = newStr.split('\n');
    final removed = [];
    final added = [];

    final oldSet = oldLines.toSet();
    final newSet = newLines.toSet();

    for (final l in oldLines) {
      if (!newSet.contains(l)) removed.add(l);
    }
    for (final l in newLines) {
      if (!oldSet.contains(l)) added.add(l);
    }
    return {'removed': removed, 'added': added};
  }
}


--------------------------------------------------------------------------------
16. Codigo: action_bar.dart
--------------------------------------------------------------------------------
/*import 'package:flutter/material.dart';

class ActionBar extends StatelessWidget {
  final VoidCallback onOpenFolder;
  final VoidCallback onSaveFile;
  final bool isEditing;

  const ActionBar({
    super.key,
    required this.onOpenFolder,
    required this.onSaveFile,
    required this.isEditing,
  });

  @override
  Widget build(BuildContext context) {
    return Container(
      height: 42,
      color: const Color(0xFF222426),
      padding: const EdgeInsets.symmetric(horizontal: 8),
      child: Row(
        children: [
          IconButton(
            icon: const Icon(Icons.folder_open, color: Colors.white70),
            tooltip: 'Abrir pasta',
            onPressed: onOpenFolder,
          ),
          IconButton(
            icon: Icon(Icons.save, color: isEditing ? Colors.greenAccent : Colors.white70),
            tooltip: 'Salvar arquivo',
            onPressed: onSaveFile,
          ),
          const Spacer(),
          const Text('MKIDEIA IDE', style: TextStyle(color: Colors.white70, fontWeight: FontWeight.bold)),
        ],
      ),
    );
  }
}*/


--------------------------------------------------------------------------------
17. Codigo: file_system_tree.dart
--------------------------------------------------------------------------------
import 'package:flutter/material.dart';
import 'file_context_menu.dart';

class FileSystemTree extends StatelessWidget {
  const FileSystemTree({super.key});

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onSecondaryTapDown: (details) {
        showDialog(
          context: context,
          builder: (_) => Stack(
            children: [
              Positioned(
                left: details.globalPosition.dx,
                top: details.globalPosition.dy,
                child: FileContextMenu(
                  isFolder: true, // ou false dependendo do nÃ³ clicado
                  onNewFile: () => print("Novo arquivo"),
                  onNewFolder: () => print("Nova pasta"),
                  onRename: () => print("Renomear"),
                  onDelete: () => print("Deletar"),
                  onCopy: () => print("Copiar"),
                  onCut: () => print("Recortar"),
                  onPaste: () => print("Colar"),
                  onDuplicate: () => print("Duplicar"),
                  onOpenInExplorer: () => print("Abrir no Explorer"),
                  onExport: () => print("Exportar"),
                ),
              ),
            ],
          ),
        );
      },
      child: const Center(
        child: Text("Clique com o botÃ£o direito aqui"),
      ),
    );
  }
}


--------------------------------------------------------------------------------
18. Codigo: file_context_menu.dart
--------------------------------------------------------------------------------
/**/
import 'package:flutter/material.dart';

class FileContextMenu extends StatelessWidget {
  final bool isFolder;
  final VoidCallback onNewFile;
  final VoidCallback onNewFolder;
  final VoidCallback onRename;
  final VoidCallback onDelete;
  final VoidCallback onCopy;
  final VoidCallback onCut;
  final VoidCallback onPaste;
  final VoidCallback onDuplicate;
  final VoidCallback onOpenInExplorer;
  final VoidCallback onExport;

  const FileContextMenu({
    super.key,
    required this.isFolder,
    required this.onNewFile,
    required this.onNewFolder,
    required this.onRename,
    required this.onDelete,
    required this.onCopy,
    required this.onCut,
    required this.onPaste,
    required this.onDuplicate,
    required this.onOpenInExplorer,
    required this.onExport,
  });

  @override
  Widget build(BuildContext context) {
    return Material(
      elevation: 8,
      borderRadius: BorderRadius.circular(8),
      child: IntrinsicWidth(
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            if (isFolder) ...[
              _buildItem(Icons.insert_drive_file, "Novo Arquivo", onNewFile, shortcut: "Ctrl+N"),
              _buildItem(Icons.create_new_folder, "Nova Pasta", onNewFolder, shortcut: "Ctrl+Shift+N"),
              const Divider(),
            ],
            _buildItem(Icons.drive_file_rename_outline, "Renomear", onRename, shortcut: "F2"),
            _buildItem(Icons.delete_outline, "Deletar", onDelete, shortcut: "Del"),
            const Divider(),
            _buildItem(Icons.copy, "Copiar", onCopy, shortcut: "Ctrl+C"),
            _buildItem(Icons.cut, "Recortar", onCut, shortcut: "Ctrl+X"),
            _buildItem(Icons.paste, "Colar", onPaste, shortcut: "Ctrl+V"),
            _buildItem(Icons.copy_all, "Duplicar", onDuplicate),
            const Divider(),
            _buildItem(Icons.folder_open, "Abrir no Explorer", onOpenInExplorer),
            _buildItem(Icons.archive_outlined, "Exportar", onExport),
          ],
        ),
      ),
    );
  }

  Widget _buildItem(IconData icon, String text, VoidCallback onTap, {String? shortcut}) {
    return InkWell(
      onTap: onTap,
      child: Padding(
        padding: const EdgeInsets.symmetric(horizontal: 12, vertical: 8),
        child: Row(
          children: [
            Icon(icon, size: 18),
            const SizedBox(width: 10),
            Expanded(child: Text(text)),
            if (shortcut != null)
              Text(shortcut, style: TextStyle(color: Colors.grey[600], fontSize: 12)),
          ],
        ),
      ),
    );
  }
}


--------------------------------------------------------------------------------
19. Codigo: language_config.dart
--------------------------------------------------------------------------------
import 'package:highlight/highlight.dart';
import 'package:highlight/languages/cpp.dart';
import 'package:highlight/languages/css.dart';
import 'package:highlight/languages/dart.dart';
import 'package:highlight/languages/java.dart';
import 'package:highlight/languages/javascript.dart';
import 'package:highlight/languages/json.dart';
import 'package:highlight/languages/markdown.dart';
import 'package:highlight/languages/php.dart';
import 'package:highlight/languages/python.dart';
import 'package:highlight/languages/xml.dart';
import 'package:highlight/languages/yaml.dart';

/// Mapeamento de extensÃµes para linguagens
final Map languageMap = {
  '.dart': dart,
  '.js': javascript,
  '.py': python,
  '.json': json,
  '.html': xml,   // HTML usa o parser XML
  '.css': css,
  '.java': java,
  '.cpp': cpp,
  '.php': php,
  '.md': markdown,
  '.yml': yaml,
  '.yaml': yaml,
  '.xml': xml,
};

/// Modo padrÃ£o para arquivos desconhecidos
final Mode plainText = Mode(
  contains: [],
  illegal: null,
);

/// Detecta a linguagem pelo caminho do arquivo
Mode getLanguageForPath(String path) {
  final ext = '.' + path.split('.').last.toLowerCase();
  return languageMap[ext] ?? plainText; // retorna plainText se nÃ£o encontrar
}


--------------------------------------------------------------------------------
20. CÃ³digo: github_theme.dart
--------------------------------------------------------------------------------
import 'package:flutter/material.dart';

// Tema para destaque de sintaxe em modo escuro
final Map githubTheme = {
  'root': const TextStyle(backgroundColor: Color(0xFF0F1115), color: Color(0xFFE6EDF3)),
  'keyword': const TextStyle(color: Color(0xFFC678DD)),
  'selector-tag': const TextStyle(color: Color(0xFFC678DD)),
  'literal': const TextStyle(color: Color(0xFFC678DD)),
  'section': const TextStyle(color: Color(0xFFC678DD)),
  'link': const TextStyle(color: Color(0xFF61AFEF)),
  'string': const TextStyle(color: Color(0xFF98C379)),
  'title': const TextStyle(color: Color(0xFF61AFEF)),
  'name': const TextStyle(color: Color(0xFF61AFEF)),
  'type': const TextStyle(color: Color(0xFF61AFEF)),
  'attribute': const TextStyle(color: Color(0xFF61AFEF)),
  'symbol': const TextStyle(color: Color(0xFF56B6C2)),
  'bullet': const TextStyle(color: Color(0xFF56B6C2)),
  'addition': const TextStyle(color: Color(0xFF56B6C2)),
  'variable': const TextStyle(color: Color(0xFF56B6C2)),
  'template-variable': const TextStyle(color: Color(0xFF56B6C2)),
  'comment': const TextStyle(color: Color(0xFF7F848E)),
  'quote': const TextStyle(color: Color(0xFF7F848E)),
  'deletion': const TextStyle(color: Color(0xFFE06C75)),
  'number': const TextStyle(color: Color(0xFFD19A66)),
  'meta': const TextStyle(color: Color(0xFFD19A66)),
  'selector-id': const TextStyle(color: Color(0xFFD19A66)),
  'selector-class': const TextStyle(color: Color(0xFFD19A66)),
  'emphasis': const TextStyle(fontStyle: FontStyle.italic),
  'strong': const TextStyle(fontWeight: FontWeight.bold),
};

// Alias explÃ­cito, caso queira referenciar diretamente
final Map githubDarkTheme = githubTheme;


--------------------------------------------------------------------------------
23. Codigo: fs_exporter.dart
--------------------------------------------------------------------------------
/*


*/

--------------------------------------------------------------------------------
23. Codigo: pubspec.yaml
--------------------------------------------------------------------------------
version: 1.0.0+1

environment:
  sdk: ^3.8.1

# Dependencies specify other packages that your package needs in order to work.
# To automatically upgrade your package dependencies to the latest versions
# consider running `flutter pub upgrade --major-versions`. Alternatively,
# dependencies can be manually updated by changing the version numbers below to
# the latest version available on pub.dev. To see which dependencies have newer
# versions available, run `flutter pub outdated`.
dependencies:
  flutter:
    sdk: flutter

  # The following adds the Cupertino Icons font to your application.
  # Use with the CupertinoIcons class for iOS style icons.
  cupertino_icons: ^1.0.8
  provider: ^6.0.5  # para gerenciamento simples de estado
  shared_preferences: ^2.1.1  # para salvar histÃ³rico localmente
  flutter_code_editor: ^0.3.0
  http: ^1.5.0
  hive: ^2.2.3
  path_provider: ^2.1.5
  riverpod: ^2.6.1
  file_picker: ^10.3.0
  highlight: ^0.7.0
  split_view: ^3.2.1
  flutter_split_view: ^0.1.2
  path: ^1.9.1

dev_dependencies:
  flutter_test:
    sdk: flutter

  # The "flutter_lints" package below contains a set of recommended lints to
  # encourage good coding practices. The lint set provided by the package is
  # activated in the `analysis_options.yaml` file located at the root of your
  # package. See that file for information about deactivating specific lint
  # rules and activating additional ones.
  flutter_lints: ^5.0.0

# For information on the generic Dart part of this file, see the
# following page: https://dart.dev/tools/pub/pubspec

# The following section is specific to Flutter packages.
flutter:

  # The following line ensures that the Material Icons font is
  # included with your application, so that you can use the icons in
  # the material Icons class.
  uses-material-design: true

  # To add assets to your application, add an assets section, like this:
  # assets:
  #   - images/a_dot_burr.jpeg
  #   - images/a_dot_ham.jpeg

  # An image asset can refer to one or more resolution-specific "variants", see
  # https://flutter.dev/to/resolution-aware-images

  # For details regarding adding assets from package dependencies, see
  # https://flutter.dev/to/asset-from-package

  # To add custom fonts to your application, add a fonts section here,
  # in this "flutter" section. Each entry in this list should have a
  # "family" key with the font family name, and a "fonts" key with a
  # list giving the asset and other descriptors for the font. For
  # example:
  # fonts:
  #   - family: Schyler
  #     fonts:
  #       - asset: fonts/Schyler-Regular.ttf
  #       - asset: fonts/Schyler-Italic.ttf
  #         style: italic
  #   - family: Trajan Pro
  #     fonts:
  #       - asset: fonts/TrajanPro.ttf
  #       - asset: fonts/TrajanPro_Bold.ttf
  #         weight: 700
  #
  # For details regarding fonts from package dependencies,
  # see https://flutter.dev/to/font-from-package







--------------------------------------------------------------------------------

--------------------------------------------------------------------------------

Pyton - Servidor

Estrutura de pasta:

mkide_ia/
    .idea/
    .venv/
    adapter/
        adapter.py
    core/
        agents/
            agent_router.py
        runners/
            shell_runner.py
        context_manager.py
        rag_indexer.py
        session_manager.py
        summarizer.py
    data/
        indexes/
        lib/
            context.json
        sessions/
            default/
                1756188647277/
                    session.jsonl
                    session.meta.json
                1756193001360/
                    session.jsonl
                    session.meta.json
                1756215353579/
                1756251367779/
                1756251693387/
                1756261072779/
                1756270671678/
                1756272539473/
                1756273876420/
                1756274612380/
            projeto_teste/
                sessÃ£o_teste/
                    session.jsonl
                    session.meta.json
    history/
    knowledge_base/
    utils/
        prompts/
        system/
            base_system.md
        io.py
        save_chat_log.py
    main.py
    mkideia.sh
    requirements.txt
    
    
--------------------------------------------------------------------------------
1. Codigo: main.py
--------------------------------------------------------------------------------    
#!/usr/bin/env python3
import os
from fastapi import FastAPI, HTTPException, Request
from fastapi.middleware.cors import CORSMiddleware
from adapter.adapter import router as adapter_router
from core.session_manager import router as session_router
from core.context_manager import router as context_router
from core.rag_indexer import router as rag_router
from core.summarizer import router as summarizer_router

app = FastAPI(title="MK IDE I.A â€” Backend")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"], allow_methods=["*"], allow_headers=["*"], allow_credentials=True
)

# Routers
app.include_router(session_router, prefix="/session", tags=["sessions"])
app.include_router(context_router, prefix="/context", tags=["context"])
app.include_router(rag_router, prefix="/rag", tags=["rag"])
app.include_router(summarizer_router, prefix="/summarize", tags=["summarize"])
app.include_router(adapter_router, prefix="", tags=["adapter"])  # mantÃ©m /v1/*

@app.get("/")
def root():
    return {"ok": True, "service": "MK IDE I.A", "endpoints": ["/session", "/v1/chat/completions"]}

if __name__ == "__main__":
    import uvicorn
    host = os.getenv("ADAPTER_HOST", "127.0.0.1")
    port = int(os.getenv("ADAPTER_PORT", "5000"))
    uvicorn.run("main:app", host=host, port=port, reload=True)


--------------------------------------------------------------------------------
2. Codigo: mkideia.sh
--------------------------------------------------------------------------------
#!/bin/bash
sudo cd "$(dirname "$0")"
source .venv/bin/activate
python main.py


--------------------------------------------------------------------------------
3. Codigo: requirements.txt
--------------------------------------------------------------------------------
fastapi==0.111.0
uvicorn[standard]==0.29.0
httpx==0.27.0
pydantic==2.7.1
python-multipart==0.0.9
orjson==3.10.7

--------------------------------------------------------------------------------
4. Codigo: adapter/adapter.py
--------------------------------------------------------------------------------
#!/usr/bin/env python3
import os, sys, time, json, logging, asyncio, subprocess
from typing import List, Any, Dict, AsyncIterator, Optional

from fastapi import APIRouter, HTTPException, Request
from fastapi.responses import StreamingResponse
from pydantic import BaseModel
import httpx
from datetime import datetime

# --- Config ---
OLLAMA_URL = os.getenv("OLLAMA_URL", "http://127.0.0.1:11434")
ADAPTER_HOST = os.getenv("ADAPTER_HOST", "127.0.0.1")
ADAPTER_PORT = int(os.getenv("ADAPTER_PORT", "5000"))

logger = logging.getLogger("adapter")
router = APIRouter()

# import dos utilitÃ¡rios de sessÃ£o
ROOT = os.getenv("MKIDE_ROOT", os.getcwd())
sys.path.insert(0, os.path.join(ROOT, "utils"))
from save_chat_log import append_message, ensure_session  # noqa

# ----- Models -----
class Message(BaseModel):
    role: str
    content: str

class ChatRequest(BaseModel):
    model: Optional[str] = None
    messages: Optional[List[Message]] = []
    stream: Optional[bool] = True
    delta_stream: Optional[bool] = False
    # sessÃ£o
    project_id: Optional[str] = "default"
    session_id: Optional[str] = None

# ----- Helpers -----
def _extract_text_and_meta(data: Any) -> (str, Dict[str, Any]):
    meta = {}
    try:
        if isinstance(data, dict):
            if "message" in data and isinstance(data["message"], dict):
                content = data["message"].get("content")
                if content:
                    meta.update({k: data.get(k) for k in ("model", "done", "done_reason", "total_duration", "load_duration") if k in data})
                    return str(content), meta
            for key in ("response", "completion", "text"):
                if key in data and data[key]:
                    meta.update({k: data.get(k) for k in ("model", "total_duration", "load_duration") if k in data})
                    return str(data[key]), meta
            if "choices" in data and isinstance(data["choices"], list) and data["choices"]:
                c0 = data["choices"][0]
                if isinstance(c0, dict):
                    msg = c0.get("message")
                    if isinstance(msg, dict) and msg.get("content"):
                        meta.update(c0)
                        return str(msg["content"]), meta
                    if c0.get("text"):
                        meta.update(c0)
                        return str(c0["text"]), meta
        return str(data), {"raw": data}
    except Exception as e:
        return str(data), {"error_extracting_meta": str(e), "raw": data}

# ----- Endpoints -----
@router.get("/v1/models")
async def v1_models():
    models = []
    async with httpx.AsyncClient(timeout=8.0) as client:
        try:
            r = await client.get(f"{OLLAMA_URL}/api/models")
            if r.status_code == 200:
                data = r.json()
                if isinstance(data, list):
                    for m in data:
                        name = m.get("name") or m.get("model") or m.get("id") or str(m)
                        models.append({"id": name, "object": "model", "owned_by": "local"})
                elif isinstance(data, dict) and "models" in data:
                    for m in data["models"]:
                        models.append({"id": m.get("name"), "object": "model", "owned_by": "local"})
                else:
                    models.append({"id": str(data), "object": "model", "owned_by": "local"})
                return {"object": "list", "data": models}
        except Exception:
            pass
    # fallback CLI
    try:
        out = subprocess.run(["ollama", "list"], capture_output=True, text=True, check=True)
        lines = out.stdout.splitlines()
        parsed = []
        for line in lines:
            line = line.strip()
            if not line or line.lower().startswith(("name", "model")):
                continue
            parts = line.split()
            if parts:
                parsed.append({"id": parts[0], "object": "model", "owned_by": "local"})
        if parsed:
            return {"object": "list", "data": parsed}
    except Exception as e:
        logger.exception("ollama list failed: %s", e)
    raise HTTPException(status_code=500, detail="NÃ£o foi possÃ­vel listar modelos.")

@router.post("/v1/chat/completions")
async def v1_chat(req: ChatRequest):
    model = (req.model or "").strip()
    if not model:
        raise HTTPException(400, "Campo 'model' Ã© obrigatÃ³rio.")

    # sessÃ£o
    project_id = req.project_id or "default"
    session_id = req.session_id
    if not session_id:
        raise HTTPException(400, "Campo 'session_id' Ã© obrigatÃ³rio. Crie em /session/new.")

    ensure_session(project_id, session_id, model=model)

    # apenda as mensagens do usuÃ¡rio no JSONL
    for m in (req.messages or []):
        if m.role == "user":
            append_message(project_id, session_id, m.role, m.content, {"model": model, "phase": "received"})

    # Real streaming com Ollama
    async def sse_stream():
        payload = {"model": model, "messages": [m.model_dump() for m in (req.messages or [])], "stream": True}
        try:
            async with httpx.AsyncClient(timeout=None) as client:
                async with client.stream("POST", f"{OLLAMA_URL}/api/chat", json=payload) as r:
                    r.raise_for_status()
                    full_text = ""
                    async for raw in r.aiter_lines():
                        if not raw:
                            continue
                        # Ollama envia linhas tipo: {"message":{"role":"assistant","content":"..."},"done":false}
                        try:
                            data = json.loads(raw)
                            if "message" in data and "content" in data["message"]:
                                delta = data["message"]["content"]
                                full_text += delta
                                yield f"data: {json.dumps({'delta': delta})}\n\n"
                            if data.get("done"):
                                break
                        except Exception:
                            # pode vir "data: ..." em alguns modos; tenta extrair
                            if raw.startswith("data:"):
                                payload = raw[len("data:"):].strip()
                                try:
                                    data = json.loads(payload)
                                    delta = data.get("message", {}).get("content", "")
                                    if delta:
                                        full_text += delta
                                        yield f"data: {json.dumps({'delta': delta})}\n\n"
                                except Exception:
                                    pass
                    # fim do stream: grava a resposta completa 1x
                    append_message(project_id, session_id, "assistant", full_text, {"model": model, "phase": "complete"})
                    yield "data: [DONE]\n\n"
        except httpx.HTTPError as e:
            logger.exception("Ollama stream error: %s", e)
            raise HTTPException(502, f"Erro no streaming do Ollama: {e}") from e

    if req.stream:
        headers = {"Cache-Control": "no-cache", "Content-Type": "text/event-stream", "Connection": "keep-alive"}
        return StreamingResponse(sse_stream(), headers=headers, status_code=200)

    # modo nÃ£o-stream (retorno Ãºnico)
    async with httpx.AsyncClient(timeout=120.0) as client:
        payload = {"model": model, "messages": [m.model_dump() for m in (req.messages or [])], "stream": False}
        r = await client.post(f"{OLLAMA_URL}/api/chat", json=payload)
        r.raise_for_status()
        data = r.json()
        text, meta = _extract_text_and_meta(data)
        append_message(project_id, session_id, "assistant", text, {"model": model, "phase": "complete", "meta": meta})
        now_ts = int(time.time())
        response_obj = {
            "id": f"chatcmpl-local-{now_ts}",
            "object": "chat.completion",
            "created": now_ts,
            "model": model,
            "choices": [{"index": 0, "message": {"role": "assistant", "content": text}, "finish_reason": "stop"}],
        }
        return response_obj


--------------------------------------------------------------------------------
5. Codigo: core/context_manager.py
--------------------------------------------------------------------------------
from fastapi import APIRouter
from pydantic import BaseModel
from utils.save_chat_log import load_tail, read_summary

router = APIRouter()

class BuildContextReq(BaseModel):
    project_id: str = "default"
    session_id: str
    max_messages: int = 30
    system_prompt: str | None = None
    task: str | None = None
    rag_chunks: list[str] | None = None

@router.post("/build")
def build_context(req: BuildContextReq):
    summary = read_summary(req.project_id, req.session_id)
    tail = load_tail(req.project_id, req.session_id, req.max_messages)
    history = [{"role": m["role"], "content": m["content"]} for m in tail]
    messages = []
    if req.system_prompt:
        messages.append({"role": "system", "content": req.system_prompt})
    if summary:
        messages.append({"role": "system", "content": f"[RESUMO DA SESSÃƒO]\n{summary}"})
    if req.task:
        messages.append({"role": "system", "content": f"[TAREFA]\n{req.task}"})
    if req.rag_chunks:
        ctx = "\n\n".join(req.rag_chunks[:6])
        messages.append({"role": "system", "content": f"[CONTEXTOS RELEVANTES]\n{ctx}"})
    messages.extend(history)
    return {"ok": True, "messages": messages}


--------------------------------------------------------------------------------
6. Codigo: core/rag_indexer.py
--------------------------------------------------------------------------------
import os, glob
from fastapi import APIRouter
from pydantic import BaseModel

router = APIRouter()

class IndexReq(BaseModel):
    project_id: str = "default"
    root_path: str

@router.post("/index")
def index_project(req: IndexReq):
    # v0: sÃ³ varre e guarda lista bÃ¡sica (melhorar depois com embeddings)
    result = []
    for path in glob.glob(os.path.join(req.root_path, "**", "*"), recursive=True):
        if os.path.isfile(path) and os.path.getsize(path) < 2_000_000:
            result.append(path)
    return {"ok": True, "files_count": len(result)}

class QueryReq(BaseModel):
    project_id: str = "default"
    root_path: str
    query: str
    limit: int = 6

@router.post("/query")
def query(req: QueryReq):
    # v0: busca burra por palavra-chave; depois trocar por embeddings
    hits = []
    for path in glob.glob(os.path.join(req.root_path, "**", "*"), recursive=True):
        if os.path.isfile(path) and os.path.getsize(path) < 500_000:
            try:
                with open(path, "r", encoding="utf-8", errors="ignore") as f:
                    txt = f.read()
                if req.query.lower() in txt.lower():
                    snippet = txt[:800]
                    hits.append({"path": path, "snippet": snippet})
            except Exception:
                pass
        if len(hits) >= req.limit: break
    return {"ok": True, "chunks": [h["snippet"] for h in hits], "hits": hits}


--------------------------------------------------------------------------------
7. Codigo: core/session_manager.py
--------------------------------------------------------------------------------
from fastapi import APIRouter, HTTPException
from pydantic import BaseModel
import uuid
from utils.save_chat_log import ensure_session

router = APIRouter()

class NewSessionReq(BaseModel):
    project_id: str = "default"
    title: str | None = None
    model: str | None = None

class SelectSessionReq(BaseModel):
    project_id: str = "default"
    session_id: str

@router.post("/new")
def new_session(req: NewSessionReq):
    session_id = uuid.uuid4().hex[:12]
    ensure_session(req.project_id, session_id, req.title, req.model)
    return {"ok": True, "project_id": req.project_id, "session_id": session_id}

@router.post("/select")
def select_session(req: SelectSessionReq):
    ensure_session(req.project_id, req.session_id)
    return {"ok": True, "project_id": req.project_id, "session_id": req.session_id}


--------------------------------------------------------------------------------
8. Codigo: core/summarizer.py
--------------------------------------------------------------------------------
from fastapi import APIRouter
from pydantic import BaseModel
import os, json

router = APIRouter()

BASE_DIR = os.path.expanduser("~/.mkide/projects")

class ContextReq(BaseModel):
    project_id: str = "default"

@router.post("/get_context")
def get_context(req: ContextReq):
    fpath = os.path.join(BASE_DIR, req.project_id, "context.json")
    if not os.path.exists(fpath):
        return {"ok": False, "context": {}}
    with open(fpath) as f:
        return {"ok": True, "context": json.load(f)}

@router.post("/set_context")
def set_context(req: ContextReq, context: dict):
    proj_dir = os.path.join(BASE_DIR, req.project_id)
    os.makedirs(proj_dir, exist_ok=True)
    fpath = os.path.join(proj_dir, "context.json")
    with open(fpath, "w") as f:
        json.dump(context, f, indent=2)
    return {"ok": True, "saved_at": fpath}


--------------------------------------------------------------------------------
9. Codigo: core/agent_router.py
--------------------------------------------------------------------------------
def choose_agent(user_text: str) -> str:
    t = user_text.lower()
    if any(k in t for k in ["docker", "deploy", "k8s", "infra"]):
        return "infra"
    if any(k in t for k in ["schema", "sql", "migration", "query"]):
        return "db"
    if any(k in t for k in ["api", "endpoint", "rest", "graphql"]):
        return "api"
    if any(k in t for k in ["flutter", "widget", "dart", "ui"]):
        return "flutter"
    return "code"  # default


--------------------------------------------------------------------------------
10. Codigo: core/runners/shell_runner.py
--------------------------------------------------------------------------------
import subprocess

def run_cmd(cmd: str, cwd: str | None = None, timeout: int = 120):
    try:
        out = subprocess.run(cmd, shell=True, cwd=cwd, capture_output=True, text=True, timeout=timeout)
        return {"ok": out.returncode == 0, "code": out.returncode, "stdout": out.stdout, "stderr": out.stderr}
    except subprocess.TimeoutExpired:
        return {"ok": False, "code": -1, "stdout": "", "stderr": "Timeout"}


--------------------------------------------------------------------------------
11. Codigo: data/indexes
--------------------------------------------------------------------------------


--------------------------------------------------------------------------------
12. Codigo: data/sessions/default
--------------------------------------------------------------------------------


--------------------------------------------------------------------------------
13. Codigo: history/
--------------------------------------------------------------------------------


--------------------------------------------------------------------------------
14. Codigo: knowledge_base/
--------------------------------------------------------------------------------


--------------------------------------------------------------------------------
15. Codigo: utils/io.py
--------------------------------------------------------------------------------
import os, json

def read_text(path: str) -> str:
    with open(path, "r", encoding="utf-8") as f:
        return f.read()

def write_text(path: str, text: str):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        f.write(text)

def write_json(path: str, data: dict):
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f, ensure_ascii=False, indent=2)


--------------------------------------------------------------------------------
16. Codigo: utils/save_chat_log.py
--------------------------------------------------------------------------------
from __future__ import annotations
from datetime import datetime
import json, os, pathlib, orjson

ROOT = os.getenv("MKIDE_ROOT", os.getcwd())
DATA_DIR = os.path.join(ROOT, "data")
SESSIONS_DIR = os.path.join(DATA_DIR, "sessions")
os.makedirs(SESSIONS_DIR, exist_ok=True)

def _session_paths(project_id: str, session_id: str):
    base = os.path.join(SESSIONS_DIR, project_id, session_id)
    os.makedirs(base, exist_ok=True)
    return {
        "base": base,
        "jsonl": os.path.join(base, "session.jsonl"),
        "meta": os.path.join(base, "session.meta.json"),
    }

def ensure_session(project_id: str, session_id: str, title: str | None = None, model: str | None = None):
    p = _session_paths(project_id, session_id)
    if not os.path.exists(p["meta"]):
        meta = {
            "project_id": project_id,
            "session_id": session_id,
            "title": title or f"SessÃ£o {session_id}",
            "model": model or "",
            "created_at": datetime.utcnow().isoformat(),
            "updated_at": datetime.utcnow().isoformat(),
            "summary": "",
            "msg_count": 0,
        }
        with open(p["meta"], "wb") as f:
            f.write(orjson.dumps(meta, option=orjson.OPT_INDENT_2))
    if not os.path.exists(p["jsonl"]):
        open(p["jsonl"], "a", encoding="utf-8").close()
    return p

def append_message(project_id: str, session_id: str, role: str, content: str, meta: dict | None = None):
    p = ensure_session(project_id, session_id)
    record = {
        "ts": datetime.utcnow().isoformat(),
        "role": role,
        "content": content,
        "meta": meta or {}
    }
    with open(p["jsonl"], "a", encoding="utf-8") as f:
        f.write(json.dumps(record, ensure_ascii=False) + "\n")
    # update meta
    with open(p["meta"], "rb") as f:
        m = orjson.loads(f.read())
    m["updated_at"] = datetime.utcnow().isoformat()
    m["msg_count"] = int(m.get("msg_count", 0)) + 1
    with open(p["meta"], "wb") as f:
        f.write(orjson.dumps(m, option=orjson.OPT_INDENT_2))
    return p

def load_tail(project_id: str, session_id: str, max_messages: int = 30):
    p = _session_paths(project_id, session_id)
    msgs = []
    if not os.path.exists(p["jsonl"]):
        return msgs
    with open(p["jsonl"], "r", encoding="utf-8") as f:
        lines = f.readlines()[-max_messages:]
    for line in lines:
        try:
            msgs.append(json.loads(line))
        except Exception:
            pass
    return msgs

def read_summary(project_id: str, session_id: str) -> str:
    p = _session_paths(project_id, session_id)
    if not os.path.exists(p["meta"]): return ""
    with open(p["meta"], "rb") as f:
        m = orjson.loads(f.read())
    return m.get("summary", "")

def write_summary(project_id: str, session_id: str, summary: str):
    p = _session_paths(project_id, session_id)
    ensure_session(project_id, session_id)
    with open(p["meta"], "rb") as f:
        m = orjson.loads(f.read())
    m["summary"] = summary
    m["updated_at"] = datetime.utcnow().isoformat()
    with open(p["meta"], "wb") as f:
        f.write(orjson.dumps(m, option=orjson.OPT_INDENT_2))


--------------------------------------------------------------------------------
16. Codigo: utils/prompts
--------------------------------------------------------------------------------



--------------------------------------------------------------------------------
16. Codigo: utils/prompts/base_system.md
--------------------------------------------------------------------------------



--------------------------------------------------------------------------------
16. Codigo: 
--------------------------------------------------------------------------------



--------------------------------------------------------------------------------
16. Codigo: 
--------------------------------------------------------------------------------



--------------------------------------------------------------------------------
16. Codigo: 
--------------------------------------------------------------------------------


PHP
Codigo:
Codigo:
Codigo:


SQL
Codigo:
Codigo:
Codigo:


Grupo 2
