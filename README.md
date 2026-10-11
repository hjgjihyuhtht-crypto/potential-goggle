# Kinect 3D Studio — Etapa 3 (sensores)

App Android que conecta um **Kinect v1 (Xbox 360)** diretamente ao celular por USB OTG,
sem PC, nuvem ou programas externos. Esta versão entrega a Etapa 3 do plano:
visualização ao vivo de profundidade, RGB e infravermelho, captura de quadros, gravação contínua
e reprodução de gravações. Formatos dos arquivos: [docs/FORMATOS.md](docs/FORMATOS.md).

## Novidades da Etapa 3

- **Profundidade bruta preservada**: o app recebe os 11 bits originais do sensor; os milímetros vêm
  da tabela de calibração gravada no próprio Kinect.
- **Tela Sensores ao vivo**: profundidade (mapa Turbo ou cinza, faixa ajustável em mm), RGB e IR,
  lado a lado ou individualmente, com FPS, contagem de quadros e defasagem entre fluxos.
- **RGB alinhado à profundidade** usando a calibração de fábrica (base para a nuvem de pontos colorida da Etapa 4).
- **Toque para medir**: distância em mm e coordenadas X, Y, Z do ponto tocado.
- **Filtros de ruído** (mediana e remoção de pixels "voadores" nas bordas). Nunca preenchem buracos.
- **Brilho e contraste** só na visualização; arquivos salvos ficam originais.
- **Capturar quadro**: profundidade bruta e em mm (PGM 16 bits), RGB/IR, RGB alinhado, calibração, prévias PNG e `metadata.json`.
- **Gravar**: fluxo contínuo em `.k3drec`, com aviso visível, controle de espaço livre e parada automática.
- **Reproduzir gravação** sem o Kinect, sempre identificada como **SIMULAÇÃO**, com pausa, velocidade, repetição e linha do tempo.
- **Capturas e gravações**: listar, reproduzir, exportar para uma pasta do celular e excluir definitivamente.


Viabilidade e decisões técnicas: [docs/ETAPA1_VIABILIDADE.md](docs/ETAPA1_VIABILIDADE.md).

## Da Etapa 2 (continua valendo)

- Detecção por VID/PID dos três sub-dispositivos do Kinect v1 e identificação do modelo (1414, 1473, Kinect for Windows) pelo `bcdDevice`.
- Kinect v2 e Azure Kinect são reconhecidos e recusados com o motivo técnico.
- Diagnóstico de energia: se só o motor/áudio aparecem, o app indica a falta da fonte de 12 V.
- Permissão USB (câmera e motor) e abertura automática quando o Android oferece o app ao plugar o Kinect.
- Inicialização real pelo libfreenect: leitura de registros e calibração da câmera.
- **Teste de fluxos**: profundidade + RGB e depois profundidade + IR, 5 s cada, medindo quadros, FPS,
  porcentagem de pixels com distância, faixa mín./máx. em mm, distância no centro e defasagem entre fluxos.
- Estimativa da banda USB isócrona a partir dos descritores.
- Motor (modelo 1414): LED, inclinação de −27° a +27° e leitura do acelerômetro.
- Tratamento de desconexão e reconexão automática.
- Log de diagnóstico copiável (inclui as mensagens do libfreenect).

Ainda **não** há visualização das imagens: isso é a Etapa 3. Nenhum botão do app executa algo simulado.

## Como compilar

### Opção A — sem computador (GitHub Actions)

1. Crie um repositório no GitHub e envie o conteúdo desta pasta (dá para fazer pelo app do GitHub ou pelo navegador do celular).
2. O workflow `.github/workflows/build-apk.yml` roda sozinho: compila o código nativo, roda os testes e gera o APK.
3. Em **Actions → execução mais recente → Artifacts**, baixe `kinect3dstudio-debug-apk` e instale no celular.

### Opção B — Android Studio

Abra a pasta no Android Studio (Ladybug ou mais recente). O Gradle baixa o NDK 27.2 e o CMake 3.22.1 se faltarem.
Ou na linha de comando: `./gradlew assembleDebug`.

## Como testar com o Kinect

1. Ligue a fonte de 12 V do Kinect na tomada e conecte o USB do Kinect ao adaptador OTG do celular.
2. O Android pergunta se quer abrir o Kinect 3D Studio: aceite (a permissão já fica concedida).
   Ou abra o app e toque em **Conectar**.
3. Com o estado "Conectado", aponte o Kinect para uma pessoa ou parede a 1–3 m e toque em **Testar fluxos**.
4. Resultado esperado: ~30 fps em profundidade, RGB e IR, e uma distância no centro coerente com a cena.
5. Se algo falhar, toque em **Copiar** no log de diagnóstico e guarde o texto para análise.

| Sintoma | Causa provável |
|---|---|
| "Câmera ausente" | Fonte de 12 V desligada ou mal encaixada |
| Conecta, mas 0 quadros | Controlador USB do celular sem suporte a isócrono de alta banda, ou hub no caminho |
| FPS baixo | Banda USB compartilhada ou CPU sob carga; tente sem hub |
| Quase nenhum pixel válido | Sensor tampado, alvo fora da faixa ~0,5–4 m ou luz solar direta |

## Estrutura

```
app/src/main/
  java/com/kinect3dstudio/
    usb/            VID/PID, descoberta, descritores, banda, UsbManager
    device/         KinectDeviceInterface, driver Kinect v1, avaliação do teste
    nativebridge/   declarações JNI
    diagnostics/    log
    ui/             Compose Material 3 (pt-BR)
  cpp/
    usb_bridge/     abertura por fd, sessão do Kinect, JNI, render em Bitmap
    sensors/        calibração, quadros, colorização, filtros, gravação/reprodução
    third_party/    libusb 1.0.27 e libfreenect (fontes completas)
    tests/          teste de fumaça nativo (host)
app/src/test/       testes JVM (30 testes)
scripts/check_native_host.sh   compila e testa o C/C++ no Linux sem NDK
docs/               viabilidade e decisões
```

## Testes

- `./gradlew testDebugUnitTest` — identificação USB, banda, parser de descritores, descoberta, avaliação de fluxos.
- `scripts/check_native_host.sh` — compila libusb + libfreenect + ponte no Linux e roda os testes nativos:
  calibração, conversão 3D, colorização, filtros, alinhamento RGB, gravação/leitura `.k3drec`
  (inclusive arquivo cortado), reprodução e captura de quadros.
- Teste de hardware: botão **Testar fluxos** no app, com o Kinect conectado.

## Licenças

- libusb: LGPL-2.1, distribuída como `libusb1.0.so` separada (substituível). Texto em `app/src/main/cpp/third_party/libusb/COPYING`.
- libfreenect: Apache-2.0 (opção escolhida do licenciamento duplo). Texto em `app/src/main/cpp/third_party/libfreenect/APACHE20`.
- Código do projeto: Apache-2.0.
