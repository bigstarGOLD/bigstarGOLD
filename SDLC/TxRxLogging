using System;
using System.IO.Ports;
using System.IO;
using System.Linq;
using System.Threading;
using System.Text;
using System.Collections.Generic;
using LSAM_DLL;

class Program
{
    static SDLC sdlcPort;
    static int rxFrameCount = 0;
    static int txFrameCount = 0;
    static object consoleLock = new object();
    static string logDirectory = "";

    static void Main(string[] args)
    {
        Console.WriteLine("╔═══════════════════════════════════════════════════╗");
        Console.WriteLine("║   SDLC RX/TX 디버깅 + 바이너리 로깅 테스트       ║");
        Console.WriteLine("╚═══════════════════════════════════════════════════╝\n");

        // SDLC 포트 목록 확인
        string[] portNames = new SDLC("INTERNAL", "INTERNAL", false, 9600, 0xFF, 0, 0).GetSDLCInfo();

        if (portNames == null || portNames.Length == 0)
        {
            Console.WriteLine("❌ SDLC 포트를 찾을 수 없습니다.");
            Console.WriteLine("   - SyncLink USB 장치가 연결되었는지 확인하세요.");
            Console.WriteLine("   - 드라이버가 설치되었는지 확인하세요.");
            PauseAndExit();
            return;
        }

        Console.WriteLine("📋 사용 가능한 SDLC 포트:");
        for (int i = 0; i < portNames.Length; i++)
        {
            Console.WriteLine($"  [{i}] {portNames[i]}");
        }

        // 포트 선택
        Console.Write("\n테스트할 포트 번호: ");
        if (!int.TryParse(Console.ReadLine(), out int idx) || idx < 0 || idx >= portNames.Length)
        {
            Console.WriteLine("❌ 잘못된 포트 번호입니다.");
            PauseAndExit();
            return;
        }

        string selectedPort = portNames[idx];

        // 테스트 모드 선택
        Console.WriteLine("\n테스트 모드 선택:");
        Console.WriteLine("  1. Loop-back 모드 (Internal Loopback)");
        Console.WriteLine("  2. External Loop-back 모드 (물리적 연결 필요)");
        Console.Write("\n선택 (1 or 2): ");
        string modeChoice = Console.ReadLine();

        bool useInternalLoopback = (modeChoice == "1");

        // SDLC 설정
        Console.WriteLine("\n⚙️  SDLC 포트 설정 중...");

        // 설정값
        string rxClock = useInternalLoopback ? "INTERNAL" : "INTERNAL";
        string txClock = "INTERNAL";
        bool loopback = useInternalLoopback;
        uint clockRate = 9600;
        uint idlePattern = 0xFF;
        uint preamblePattern = 0;
        uint preambleBit = 8;

        sdlcPort = new SDLC(rxClock, txClock, loopback, clockRate, idlePattern, preamblePattern, preambleBit);

        Console.WriteLine($"   RX Clock: {rxClock}");
        Console.WriteLine($"   TX Clock: {txClock}");
        Console.WriteLine($"   Loopback: {loopback}");
        Console.WriteLine($"   Clock Rate: {clockRate} bps");
        Console.WriteLine($"   Idle Pattern: 0x{idlePattern:X2}");
        Console.WriteLine($"   Preamble: Pattern=0x{preamblePattern:X2}, Bits={preambleBit}");

        // ============= 바이너리 로깅 시작 =============
        logDirectory = Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "SDLC_Logs");
        SDLC.StartLogging(logDirectory);

        Console.WriteLine($"\n📝 바이너리 로그 시작!");
        Console.WriteLine($"   디렉토리: {logDirectory}");
        Console.WriteLine($"   파일명: SDLC_YYMMDD_HHMMSS.bin");
        Console.WriteLine($"   포맷: 바이너리 (TX/RX 타임스탬프 포함)");
        // ============= 바이너리 로깅 시작 끝 =============

        // RX 이벤트 핸들러 등록
        sdlcPort.p.MyEvent += OnReceiveData;
        Console.WriteLine("✅ RX 이벤트 핸들러 등록 완료");

        // 포트 열기
        Console.WriteLine($"\n📡 포트 열기: {selectedPort}");
        if (!sdlcPort.Open(selectedPort))
        {
            Console.WriteLine("❌ 포트 열기 실패!");
            SDLC.StopLogging();
            PauseAndExit();
            return;
        }

        Console.WriteLine("✅ 포트 열기 성공!");
        Console.WriteLine($"   IsOpen: {sdlcPort.IsOpen()}");

        // 테스트 안내
        if (useInternalLoopback)
        {
            Console.WriteLine("\n🔄 Internal Loop-back 모드");
            Console.WriteLine("   소프트웨어적으로 TX → RX 자동 연결");
        }
        else
        {
            Console.WriteLine("\n🔌 External Loop-back 모드");
            Console.WriteLine("   ⚠️  하드웨어 연결 확인:");
            Console.WriteLine("   - TXD (PIN 2) ↔ RXD (PIN 3)");
            Console.WriteLine("   - TXC (PIN 15) ↔ RXC (PIN 17)");
        }

        Console.WriteLine("\n🚀 테스트 시작...");
        Console.WriteLine("   'q' 키를 누르면 종료합니다.");
        Console.WriteLine("   모든 TX/RX 데이터가 바이너리 파일에 기록됩니다.\n");
        Console.WriteLine(new string('═', 80));

        // 송신 스레드 시작
        Thread txThread = new Thread(TransmitThread);
        txThread.IsBackground = true;
        txThread.Start();

        // 통계 출력 스레드
        Thread statsThread = new Thread(StatisticsThread);
        statsThread.IsBackground = true;
        statsThread.Start();

        // 종료 대기
        while (true)
        {
            if (Console.KeyAvailable && Console.ReadKey(true).KeyChar == 'q')
            {
                break;
            }
            Thread.Sleep(100);
        }

        // 종료 처리
        Console.WriteLine("\n\n⏹️  테스트 종료 중...");

        Thread.Sleep(1000); // 마지막 데이터 수신 대기

        sdlcPort.Close();

        // ============= 바이너리 로깅 종료 =============
        SDLC.StopLogging();
        Console.WriteLine("📝 바이너리 로그 파일 닫기 완료");
        // ============= 바이너리 로깅 종료 끝 =============

        // 최종 통계
        Console.WriteLine("\n" + new string('═', 80));
        Console.WriteLine("📊 최종 통계");
        Console.WriteLine(new string('═', 80));
        Console.WriteLine($"총 송신 프레임: {txFrameCount}");
        Console.WriteLine($"총 수신 프레임: {rxFrameCount}");

        if (txFrameCount > 0)
        {
            double successRate = (double)rxFrameCount / txFrameCount * 100;
            Console.WriteLine($"수신 성공률: {successRate:F2}%");

            if (successRate >= 99)
            {
                Console.WriteLine("\n✅ 완벽한 Loop-back 통신!");
            }
            else if (successRate >= 90)
            {
                Console.WriteLine("\n⚠️  대부분 성공 (일부 프레임 손실)");
            }
            else if (successRate >= 50)
            {
                Console.WriteLine("\n⚠️  불안정한 통신 (하드웨어 연결 확인 필요)");
            }
            else
            {
                Console.WriteLine("\n❌ 통신 실패 (연결 또는 설정 문제)");
            }
        }

        Console.WriteLine(new string('═', 80));

        // 로그 파일 위치 안내
        Console.WriteLine($"\n📄 바이너리 로그 저장 위치:");
        Console.WriteLine($"   {logDirectory}");

        // 생성된 로그 파일 찾기
        if (Directory.Exists(logDirectory))
        {
            string[] logFiles = Directory.GetFiles(logDirectory, "SDLC_*.bin");
            if (logFiles.Length > 0)
            {
                // 가장 최근 파일
                string latestFile = logFiles[logFiles.Length - 1];
                FileInfo fi = new FileInfo(latestFile);

                Console.WriteLine($"\n   최신 로그 파일:");
                Console.WriteLine($"   - 파일명: {fi.Name}");
                Console.WriteLine($"   - 크기: {fi.Length:N0} bytes");
                Console.WriteLine($"   - 생성시간: {fi.CreationTime:yyyy-MM-dd HH:mm:ss}");

                Console.WriteLine($"\n💡 이 파일을 분석하려면:");
                Console.WriteLine($"   - 바이너리 에디터로 열기");
                Console.WriteLine($"   - 또는 별도의 로그 분석 프로그램 사용");
            }
        }

        PauseAndExit();
    }

    /// <summary>
    /// RX 데이터 수신 이벤트 핸들러
    /// </summary>
    static void OnReceiveData(object sender, byte[] data, int datalen)
    {
        rxFrameCount++;

        lock (consoleLock)
        {
            Console.ForegroundColor = ConsoleColor.Green;
            Console.WriteLine($"\n[RX #{rxFrameCount}] {DateTime.Now:HH:mm:ss.fff} 💾 바이너리 로그 기록됨");
            Console.ResetColor();

            Console.WriteLine($"  길이: {datalen} bytes");

            // HEX 출력
            Console.Write($"  HEX:  ");
            for (int i = 0; i < Math.Min(datalen, 32); i++)
            {
                Console.Write($"{data[i]:X2} ");
                if ((i + 1) % 16 == 0 && i < datalen - 1)
                    Console.Write("\n        ");
            }
            if (datalen > 32)
                Console.Write($"... (총 {datalen} bytes)");
            Console.WriteLine();

            // ASCII 출력 (출력 가능한 문자만)
            Console.Write($"  ASCII: ");
            for (int i = 0; i < Math.Min(datalen, 32); i++)
            {
                char c = (char)data[i];
                if (c >= 32 && c < 127)
                    Console.Write(c);
                else
                    Console.Write('.');
            }
            if (datalen > 32)
                Console.Write("...");
            Console.WriteLine();

            // 프레임 구조 분석 (HDLC 가정)
            if (datalen >= 3)
            {
                Console.WriteLine($"  구조:");
                Console.WriteLine($"    Address: 0x{data[0]:X2}");
                Console.WriteLine($"    Control: 0x{data[1]:X2}");
                if (datalen > 3)
                {
                    Console.WriteLine($"    Data: {datalen - 3} bytes");
                }
            }

            Console.WriteLine(new string('─', 80));
        }

        // ✅ WriteLogRecord는 SDLC.cs의 ReceiveFunction에서 자동 호출됨!
    }

    /// <summary>
    /// 송신 스레드 - 주기적으로 데이터 전송
    /// </summary>
    static void TransmitThread()
    {
        Thread.Sleep(1000); // 초기 대기

        while (true)
        {
            try
            {
                txFrameCount++;

                // 테스트 데이터 생성
                string message = $"TEST_{txFrameCount:D4}";
                byte[] frame = CreateSDLCFrame(0x01, 0x00, Encoding.ASCII.GetBytes(message));

                // 전송
                bool success = sdlcPort.WriteData(frame);

                lock (consoleLock)
                {
                    Console.ForegroundColor = ConsoleColor.Cyan;
                    Console.WriteLine($"\n[TX #{txFrameCount}] {DateTime.Now:HH:mm:ss.fff} 💾 바이너리 로그 기록됨");
                    Console.ResetColor();

                    Console.WriteLine($"  메시지: {message}");
                    Console.WriteLine($"  길이: {frame.Length} bytes");
                    Console.Write($"  HEX: ");
                    for (int i = 0; i < Math.Min(frame.Length, 20); i++)
                    {
                        Console.Write($"{frame[i]:X2} ");
                    }
                    if (frame.Length > 20)
                        Console.Write("...");
                    Console.WriteLine();
                    Console.WriteLine($"  결과: {(success ? "✅ 성공" : "❌ 실패")}");
                    Console.WriteLine(new string('─', 80));
                }

                Thread.Sleep(2000); // 2초마다 전송

                // ✅ WriteLogRecord는 SDLC.cs의 WriteData에서 자동 호출됨!
            }
            catch (ThreadAbortException)
            {
                break;
            }
            catch (Exception ex)
            {
                lock (consoleLock)
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine($"[TX ERROR] {ex.Message}");
                    Console.ResetColor();
                }
            }
        }
    }

    /// <summary>
    /// 통계 출력 스레드
    /// </summary>
    static void StatisticsThread()
    {
        while (true)
        {
            Thread.Sleep(5000); // 5초마다

            lock (consoleLock)
            {
                Console.ForegroundColor = ConsoleColor.Yellow;
                Console.WriteLine($"\n[통계] {DateTime.Now:HH:mm:ss}");
                Console.ResetColor();
                Console.WriteLine($"  TX: {txFrameCount} 프레임");
                Console.WriteLine($"  RX: {rxFrameCount} 프레임");

                if (txFrameCount > 0)
                {
                    double rate = (double)rxFrameCount / txFrameCount * 100;
                    Console.WriteLine($"  성공률: {rate:F1}%");
                }

                // Activity 체크 (최근 2초 이내 데이터 있는지)
                bool isActive = SDLC.IsActive(2);
                Console.WriteLine($"  활성 상태: {(isActive ? "✅" : "❌")}");
                Console.WriteLine($"  💾 바이너리 로그: 실시간 기록 중...");

                Console.WriteLine(new string('─', 80));
            }
        }
    }

    /// <summary>
    /// SDLC 프레임 생성 (간단한 HDLC 프레임)
    /// </summary>
    static byte[] CreateSDLCFrame(byte address, byte control, byte[] data)
    {
        List<byte> frame = new List<byte>();

        frame.Add(address);
        frame.Add(control);

        if (data != null && data.Length > 0)
        {
            frame.AddRange(data);
        }

        return frame.ToArray();
    }

    static void PauseAndExit()
    {
        Console.WriteLine("\n아무 키나 눌러 종료...");
        Console.ReadKey();
    }
}
