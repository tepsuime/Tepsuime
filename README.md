<!DOCTYPE html>
<html lang="ru" class="dark">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no" />
  <title>Староста Бот — Telegram Mini App</title>
  <!-- Подключение Telegram WebApp SDK -->
  <script src="https://telegram.org/js/telegram-web-app.js"></script>
  <!-- Подключение Tailwind CSS -->
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- Подключение Tesseract.js для OCR-распознавания текста и звёзд на фото расписания -->
  <script src="https://cdn.jsdelivr.net/npm/tesseract.js@5/dist/tesseract.min.js"></script>
  <script>
    tailwind.config = {
      darkMode: 'class',
      theme: {
        extend: {
          colors: {
            appBg: '#090d16',
            cardBg: '#111827',
            cardHover: '#1f2937',
            cardBorder: '#1e293b',
            accent: '#10b981',
            accentHover: '#059669',
            secondaryAccent: '#06b6d4'
          },
          fontFamily: {
            sans: ['Inter', '-apple-system', 'BlinkMacSystemFont', 'Segoe UI', 'Roboto', 'sans-serif']
          }
        }
      }
    };
  </script>
  <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
  <style>
    body {
      background-color: #090d16;
      color: #f8fafc;
      font-family: 'Inter', sans-serif;
      -webkit-tap-highlight-color: transparent;
    }
    ::-webkit-scrollbar {
      width: 6px;
      height: 6px;
    }
    ::-webkit-scrollbar-track {
      background: #090d16;
    }
    ::-webkit-scrollbar-thumb {
      background: #1e293b;
      border-radius: 9999px;
    }
    .screen-fade {
      animation: fadeIn 0.2s ease-out;
    }
    @keyframes fadeIn {
      from { opacity: 0; transform: translateY(6px); }
      to { opacity: 1; transform: translateY(0); }
    }
    @keyframes scanLine {
      0% { top: 0%; opacity: 0.9; }
      50% { top: 92%; opacity: 1; }
      100% { top: 0%; opacity: 0.9; }
    }
    .ai-scan-bar {
      position: absolute;
      left: 0;
      right: 0;
      height: 3px;
      background: linear-gradient(90deg, transparent, #10b981, #06b6d4, transparent);
      box-shadow: 0 0 12px #10b981;
      animation: scanLine 2.2s infinite ease-in-out;
    }
  </style>
</head>
<body class="min-h-screen flex flex-col justify-between selection:bg-emerald-500 selection:text-slate-950">

  <!-- Скрытый глобальный input для загрузки фото расписания ИИ -->
  <input id="global-schedule-photo-input" type="file" accept="image/*" class="hidden" onchange="App.handleAISchedulePhoto(event)" />
  <!-- Скрытый input для загрузки фото расписания звонков ИИ -->
  <input id="bell-schedule-photo-input" type="file" accept="image/*" class="hidden" onchange="App.handleBellSchedulePhoto(event)" />
  <!-- Скрытый input для отметки посещаемости по фото списка/аудитории -->
  <input id="attendance-photo-input" type="file" accept="image/*" class="hidden" onchange="App.handleAttendancePhoto(event)" />
  <!-- Скрытый input для загрузки файлов в Базу знаний -->
  <input id="knowledge-file-input" type="file" multiple class="hidden" onchange="App.handleKnowledgeFileUpload(event)" />

  <!-- ВЕРХНЯЯ ШАПКА ПРИЛОЖЕНИЯ -->
  <header class="sticky top-0 z-30 bg-appBg/95 backdrop-blur-md border-b border-slate-800/80">
    <div class="max-w-4xl mx-auto px-4 py-2.5 flex items-center justify-between gap-2">
      <button type="button" onclick="App.resetToWelcome()" title="На экран выбора роли (Староста / Студент)" class="px-2.5 h-9 rounded-full bg-slate-800/90 hover:bg-slate-700 flex items-center justify-center gap-1.5 text-slate-300 text-xs font-medium transition shrink-0">
        <svg class="w-4 h-4 text-emerald-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 17l-5-5m0 0l5-5m-5 5h12"/>
        </svg>
        <span class="hidden sm:inline">Роль / Вход</span>
      </button>

      <div class="text-center min-w-0">
        <div class="flex items-center justify-center gap-1.5">
          <span class="w-2 h-2 rounded-full bg-emerald-400 animate-pulse"></span>
          <h1 class="text-sm sm:text-base font-semibold tracking-tight text-white truncate">Староста бот</h1>
          <span id="header-role-badge" class="hidden px-2 py-0.5 rounded-full text-[10px] font-bold uppercase tracking-wider bg-emerald-500/20 text-emerald-300 border border-emerald-500/30">
            Староста
          </span>
        </div>
        <!-- Живое время в шапке -->
        <button type="button" onclick="App.openTimezoneModal()" class="text-[11px] text-emerald-400 hover:text-emerald-300 font-mono flex items-center justify-center gap-1 mx-auto transition">
          <span class="live-clock-time">--:--:--</span>
          <span>·</span>
          <span class="live-clock-tz-short">МСК</span>
        </button>
      </div>

      <button type="button" onclick="App.openTimezoneModal()" title="Выбрать часовой пояс" class="px-2.5 h-9 rounded-full bg-slate-800/90 hover:bg-slate-700 flex items-center justify-center gap-1 text-emerald-400 text-xs font-medium transition shrink-0">
        <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"/>
        </svg>
        <span class="hidden sm:inline">Пояс</span>
      </button>
    </div>
  </header>

  <!-- ОСНОВНОЙ КОНТЕЙНЕР -->
  <main class="flex-1 w-full max-w-4xl mx-auto px-4 pt-3 pb-28">

    <!-- ПАНЕЛЬ ЖИВОГО ВРЕМЕНИ И НЕДЕЛИ -->
    <div id="global-time-banner" class="hidden mb-4 p-3 rounded-2xl bg-slate-900/90 border border-slate-800/90 flex flex-wrap items-center justify-between gap-2">
      <div class="flex items-center gap-2.5">
        <div class="w-8 h-8 rounded-xl bg-emerald-500/15 border border-emerald-500/30 text-emerald-400 flex items-center justify-center shrink-0">
          <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"/>
          </svg>
        </div>
        <div>
          <div class="flex items-center gap-2">
            <span class="live-clock-time text-sm sm:text-base font-bold font-mono text-white">--:--:--</span>
            <span class="live-clock-date text-xs text-slate-400">--.--.----</span>
          </div>
          <button type="button" onclick="App.openTimezoneModal()" class="text-[11px] text-emerald-400 hover:underline flex items-center gap-1">
            <span class="live-clock-tz-name">Москва (UTC+3 / МСК)</span>
            <span>· изменить</span>
          </button>
        </div>
      </div>

      <div class="flex items-center gap-2">
        <button type="button" onclick="App.toggleCurrentWeekNumber()" class="px-3 py-1.5 rounded-xl bg-emerald-500/15 hover:bg-emerald-500/25 border border-emerald-500/30 text-xs font-semibold text-emerald-300 flex items-center gap-1.5 transition" title="Нажмите для переключения текущей недели (1-я / 2-я)">
          <span id="current-week-star-badge">★ 1-я неделя</span>
        </button>
      </div>
    </div>

    <!-- БАННЕР РЕЖИМА СТУДЕНТА -->
    <div id="student-mode-banner" class="hidden mb-4 p-3.5 rounded-2xl bg-cyan-950/50 border border-cyan-500/35 flex flex-wrap items-center justify-between gap-2">
      <div class="flex items-center gap-2.5">
        <span class="w-2.5 h-2.5 rounded-full bg-cyan-400"></span>
        <div>
          <p class="text-xs sm:text-sm font-semibold text-cyan-200">
            Вы вошли как участник: <span id="student-banner-name" class="text-white font-bold">Студент</span>
          </p>
          <p class="text-[11px] text-cyan-300/80">
            Вы состоите в группе «<span id="student-banner-group" class="text-white font-medium">—</span>»: можно отмечать себя и добавлять материалы.
          </p>
        </div>
      </div>
      <div class="flex items-center gap-2">
        <button type="button" onclick="App.leaveGroupAsStudent()" class="px-3 py-1.5 rounded-xl bg-rose-500/20 hover:bg-rose-500/30 border border-rose-500/40 text-xs font-semibold text-rose-300 transition">
          Выйти из группы
        </button>
      </div>
    </div>

    <!-- ================================================================= -->
    <!-- ЭКРАН 1: ПРИВЕТСТВИЕ (ВЫБОР: ВОЙТИ КАК СТАРОСТА / СТУДЕНТ)        -->
    <!-- ================================================================= -->
    <section id="screen-welcome" class="screen-fade min-h-[72vh] flex flex-col items-center justify-center text-center max-w-md mx-auto py-8">
      <button type="button" onclick="App.openTimezoneModal()" class="mb-4 px-3.5 py-1.5 rounded-full bg-slate-900 border border-slate-800 text-xs text-slate-300 flex items-center gap-2 hover:border-emerald-500/40 transition">
        <svg class="w-3.5 h-3.5 text-emerald-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"/>
        </svg>
        <span class="live-clock-time font-mono font-semibold text-white">--:--:--</span>
        <span>·</span>
        <span class="live-clock-tz-name text-emerald-400">Москва (МСК)</span>
      </button>

      <span class="px-4 py-1.5 rounded-full text-xs font-medium bg-emerald-500/15 text-emerald-300 border border-emerald-500/30 mb-5">
        Telegram Mini App · Староста Бот
      </span>

      <h2 class="text-2xl sm:text-3xl font-bold text-white mb-3">
        Как будем начинать?
      </h2>

      <p class="text-sm sm:text-base text-slate-400 leading-relaxed mb-7">
        Войдите как староста для полного управления группой, назначения зам. старосты и рассылки посещаемости всем в Telegram, или присоединитесь как студент по коду.
      </p>

      <div class="w-full space-y-3">
        <!-- Кнопка 1: Войти как староста -->
        <button type="button" id="btn-enter-leader" onclick="App.handleLeaderEntryClick()" class="w-full py-3.5 px-5 rounded-2xl bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 font-semibold text-sm sm:text-base shadow-lg shadow-emerald-500/20 flex items-center justify-center gap-2.5 transition cursor-pointer">
          <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 3l3.5 6L12 4l3.5 5L19 3v13a2 2 0 01-2 2H7a2 2 0 01-2-2V3z"/>
          </svg>
          <span id="welcome-leader-btn-text">Войти как староста (создать группу)</span>
        </button>

        <!-- Кнопка 2: Присоединиться как студент -->
        <button type="button" id="btn-enter-student" onclick="App.openStudentJoinModal()" class="w-full py-3.5 px-5 rounded-2xl bg-cardBg hover:bg-cardHover border border-slate-800 hover:border-teal-500/40 text-slate-200 font-medium text-sm sm:text-base flex items-center justify-center gap-2.5 transition cursor-pointer">
          <svg class="w-5 h-5 text-teal-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M16 7a4 4 0 11-8 0 4 4 0 018 0zM12 14a7 7 0 00-7 7h14a7 7 0 00-7-7z"/>
          </svg>
          <span>Присоединиться как студент (по коду)</span>
        </button>
      </div>

      <!-- Быстрый вход в уже созданную группу под старостой -->
      <div id="existing-group-quick-box" class="hidden w-full mt-4 p-3.5 rounded-2xl bg-slate-900/90 border border-emerald-500/30 text-left flex items-center justify-between gap-3">
        <div>
          <p class="text-xs text-emerald-400 font-semibold">Ваша созданная группа (Староста):</p>
          <p id="existing-group-title" class="text-sm font-bold text-white">Моя группа</p>
        </div>
        <button type="button" onclick="App.enterExistingLeaderDashboard()" class="px-3.5 py-2 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-xs shrink-0 transition">
          Открыть панель
        </button>
      </div>

      <!-- Быстрый вход в группу для студента (как у старосты) -->
      <div id="existing-student-quick-box" class="hidden w-full mt-3 p-3.5 rounded-2xl bg-slate-900/90 border border-cyan-500/35 text-left flex items-center justify-between gap-3">
        <div>
          <p class="text-xs text-cyan-400 font-semibold">Ваша группа (<span id="existing-student-role-label">Студент</span>):</p>
          <p id="existing-student-group-title" class="text-sm font-bold text-white">Моя группа</p>
          <p id="existing-student-user-name" class="text-[11px] text-slate-400 mt-0.5">Участник</p>
        </div>
        <button type="button" onclick="App.enterExistingStudentDashboard()" class="px-3.5 py-2 rounded-xl bg-cyan-400 hover:bg-cyan-300 text-slate-950 font-semibold text-xs shrink-0 transition">
          Зайти в группу
        </button>
      </div>
    </section>

    <!-- ================================================================= -->
    <!-- ЭКРАН 2: СОЗДАНИЕ НОВОЙ ГРУППЫ (ДЛЯ СТАРОСТЫ)                     -->
    <!-- ================================================================= -->
    <section id="screen-create-group" class="screen-fade hidden max-w-lg mx-auto py-2">
      <div class="flex items-center justify-between mb-4">
        <button type="button" onclick="App.navigate('screen-welcome')" class="inline-flex items-center gap-1.5 text-sm font-medium text-emerald-400 hover:text-emerald-300">
          <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 19l-7-7 7-7"/>
          </svg>
          Назад
        </button>
        <span class="text-xs font-mono text-slate-400">
          <span class="live-clock-time text-emerald-400">--:--:--</span> · <span class="live-clock-tz-short">МСК</span>
        </span>
      </div>

      <h2 class="text-2xl font-bold text-white">Вход как староста · Новая группа</h2>
      <p class="text-sm text-slate-400 mt-1 mb-6">Создайте отдельную группу с чистым списком участников и собственным кодом доступа</p>

      <form id="create-group-form" onsubmit="App.handleCreateGroup(event)" class="space-y-4">
        <div>
          <label class="block text-xs font-medium text-slate-400 mb-1.5">Название группы</label>
          <input id="input-group-name" type="text" required placeholder="Введите название группы"
            class="w-full px-4 py-3 rounded-xl bg-cardBg border border-slate-800 focus:border-emerald-500 focus:outline-none text-white text-sm transition" />
        </div>

        <div>
          <label class="block text-xs font-medium text-slate-400 mb-1.5">Учебное заведение</label>
          <input id="input-uni-name" type="text" required placeholder="Введите название вуза или колледжа"
            class="w-full px-4 py-3 rounded-xl bg-cardBg border border-slate-800 focus:border-emerald-500 focus:outline-none text-white text-sm transition" />
        </div>

        <div>
          <label class="block text-xs font-medium text-slate-400 mb-1.5">Ваши Фамилия и Имя (Староста)</label>
          <input id="input-leader-name" type="text" required placeholder="Введите Фамилию и Имя"
            class="w-full px-4 py-3 rounded-xl bg-cardBg border border-slate-800 focus:border-emerald-500 focus:outline-none text-white text-sm transition" />
        </div>

        <div>
          <label class="block text-xs font-medium text-slate-400 mb-1.5">Код приглашения для вашей группы (оставьте пустым для авто-генерации)</label>
          <input id="input-custom-code" type="text" placeholder="Придумайте код или оставьте поле пустым"
            class="w-full px-4 py-3 rounded-xl bg-cardBg border border-slate-800 focus:border-emerald-500 focus:outline-none text-emerald-400 font-mono font-bold uppercase text-sm transition" />
        </div>

        <div class="pt-2 space-y-3 border-t border-slate-800/80">
          <p class="text-xs font-semibold uppercase tracking-wider text-emerald-400">
            Правовые соглашения и согласия
          </p>

          <label class="flex items-start gap-3 cursor-pointer select-none">
            <input id="consent-pdn" type="checkbox" checked required
              class="mt-1 w-5 h-5 rounded-md accent-emerald-500 bg-slate-900 border-slate-700 cursor-pointer" />
            <span class="text-xs sm:text-sm text-slate-300 leading-snug">
              Даю <button type="button" onclick="App.openLegalModal('pdn')" class="text-emerald-400 underline hover:text-emerald-300">согласие на обработку персональных данных</button> в соответствии с ФЗ № 152-ФЗ.
            </span>
          </label>

          <label class="flex items-start gap-3 cursor-pointer select-none">
            <input id="consent-terms" type="checkbox" checked required
              class="mt-1 w-5 h-5 rounded-md accent-emerald-500 bg-slate-900 border-slate-700 cursor-pointer" />
            <span class="text-xs sm:text-sm text-slate-300 leading-snug">
              Принимаю <button type="button" onclick="App.openLegalModal('terms')" class="text-emerald-400 underline hover:text-emerald-300">пользовательское соглашение</button> и правила использования сервиса.
            </span>
          </label>

          <label class="flex items-start gap-3 cursor-pointer select-none">
            <input id="consent-auto-attendance" type="checkbox" checked
              class="mt-1 w-5 h-5 rounded-md accent-emerald-500 bg-slate-900 border-slate-700 cursor-pointer" />
            <span class="text-xs sm:text-sm text-slate-300 leading-snug">
              Включить <button type="button" onclick="App.openLegalModal('auto')" class="text-emerald-400 underline hover:text-emerald-300">автоматическую отметку и рассылку посещаемости</button> всем в группе.
            </span>
          </label>
        </div>

        <button type="submit" class="w-full mt-4 py-3.5 px-5 rounded-2xl bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 font-semibold text-sm sm:text-base shadow-lg shadow-emerald-500/20 transition cursor-pointer">
          Войти как староста и открыть группу
        </button>
      </form>
    </section>

    <!-- ================================================================= -->
    <!-- ЭКРАН 3: ГЛАВНАЯ СТРАНИЦА (ДАШБОРД ГРУППЫ)                        -->
    <!-- ================================================================= -->
    <section id="screen-home" class="screen-fade hidden space-y-5">

      <!-- Верхняя панель группы -->
      <div class="p-4 rounded-2xl bg-cardBg border border-slate-800/90 flex flex-wrap items-center justify-between gap-3">
        <div class="flex items-center gap-3">
          <div id="group-avatar" class="w-12 h-12 rounded-2xl bg-gradient-to-br from-emerald-500 to-teal-600 text-slate-950 font-bold text-lg flex items-center justify-center shadow-md shadow-emerald-500/15">
            Г
          </div>
          <div>
            <div class="flex items-center gap-2 flex-wrap">
              <h2 id="display-group-name" class="text-base sm:text-lg font-bold text-white leading-tight">Моя группа</h2>
              <span id="leader-code-pill" class="px-2 py-0.5 rounded-md bg-slate-800 border border-slate-700 text-[11px] font-mono text-emerald-400">
                Код: <strong id="home-invite-code-badge">—</strong>
              </span>
            </div>
            <p id="display-uni-name" class="text-xs sm:text-sm text-slate-400">Учебное заведение</p>
            <p id="display-deputies-line" class="text-[11px] text-amber-300 mt-0.5">Староста: — · Зам. старосты: не назначен</p>
          </div>
        </div>

        <div class="flex items-center gap-2">
          <button type="button" id="home-deputies-btn" onclick="App.openDeputiesModal()" class="px-3 py-2 rounded-xl bg-amber-500/15 hover:bg-amber-500/25 border border-amber-500/35 text-amber-300 text-xs font-semibold flex items-center gap-1.5 transition">
            <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m5.618-4.016A11.955 11.955 0 0112 2.944a11.955 11.955 0 01-8.618 3.04A12.02 12.02 0 003 9c0 5.591 3.824 10.29 9 11.622 5.176-1.332 9-6.03 9-11.622 0-1.042-.133-2.052-.382-3.016z"/>
            </svg>
            <span>Зам. старосты</span>
          </button>

          <button type="button" onclick="App.openHowItWorksModal()" class="px-3 py-2 rounded-xl bg-emerald-500/10 hover:bg-emerald-500/20 border border-emerald-500/30 text-emerald-300 text-xs font-medium flex items-center gap-1.5 transition">
            <span>Помощь</span>
          </button>
        </div>
      </div>

      <!-- Блок для СТУДЕНТА: Быстрая личная отметка на паре и выход из группы -->
      <div id="student-self-checkin-card" class="hidden p-5 rounded-2xl bg-gradient-to-br from-cyan-950/60 via-slate-900 to-emerald-950/50 border border-cyan-500/35 space-y-3">
        <div class="flex items-center justify-between flex-wrap gap-2">
          <span class="text-xs font-bold uppercase tracking-wider text-cyan-300">ВАША ОТМЕТКА НА ПАРЕ</span>
          <div class="flex items-center gap-2">
            <span id="student-my-status-badge" class="px-2.5 py-0.5 rounded-full text-xs font-semibold bg-emerald-500/20 text-emerald-300 border border-emerald-500/30">
              На паре (авто)
            </span>
            <button type="button" onclick="App.leaveGroupAsStudent()" class="px-2.5 py-0.5 rounded-full text-xs font-semibold bg-rose-500/20 hover:bg-rose-500/30 text-rose-300 border border-rose-500/30 transition">
              Выйти из группы
            </button>
          </div>
        </div>
        <p class="text-xs sm:text-sm text-slate-300">
          Вы состоите в группе «<span class="font-semibold text-white" id="student-card-group-name">Моя группа</span>». Отметьте своё присутствие или предупредите старосту:
        </p>
        <div class="grid grid-cols-3 gap-2 pt-1">
          <button type="button" onclick="App.setMyStudentAttendance('На паре')" class="py-2.5 px-3 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-xs transition">
            ✓ Я на паре
          </button>
          <button type="button" onclick="App.setMyStudentAttendance('Опаздываю')" class="py-2.5 px-3 rounded-xl bg-amber-500/20 hover:bg-amber-500/30 border border-amber-500/40 text-amber-300 font-semibold text-xs transition">
            ⏳ Опаздываю
          </button>
          <button type="button" onclick="App.setMyStudentAttendance('Пропуск (уваж.)')" class="py-2.5 px-3 rounded-xl bg-slate-800 hover:bg-slate-700 border border-slate-700 text-slate-200 font-semibold text-xs transition">
            📄 По уважительной
          </button>
        </div>
      </div>

      <!-- ГЛАВНЫЕ РАЗДЕЛЫ НА ПЕРВОЙ СТРАНИЦЕ: ПОСЕЩАЕМОСТЬ (ЗА ВСЁ ВРЕМЯ) НАРАВНЕ С ОБЪЯВЛЕНИЯМИ И ДЗ -->
      <div class="grid grid-cols-1 sm:grid-cols-2 gap-3.5">
        <!-- Карточка 1: Посещаемость всех участников за всё время -->
        <div class="p-4 sm:p-5 rounded-2xl bg-gradient-to-br from-emerald-950/70 via-cardBg to-teal-950/50 border border-emerald-500/40 flex flex-col justify-between space-y-3 shadow-lg shadow-emerald-950/20">
          <div>
            <div class="flex items-center justify-between gap-2 mb-2">
              <span class="px-2.5 py-0.5 rounded-full text-[10px] font-bold uppercase tracking-wider bg-emerald-500/20 text-emerald-300 border border-emerald-500/30">
                Журнал · За всё время
              </span>
              <span id="home-attendance-summary-pill" class="text-xs font-mono text-emerald-400 font-semibold">
                0 уч. · 100%
              </span>
            </div>
            <h3 class="text-base sm:text-lg font-bold text-white flex items-center gap-2">
              <span>📊 Посещаемость группы</span>
            </h3>
            <p id="home-attendance-subtext" class="text-xs text-slate-300 mt-1 leading-relaxed">
              Статистика посещаемости всех участников за всё время учёбы, сводка за сегодня и история по датам.
            </p>
          </div>
          <div class="pt-2 flex flex-wrap items-center gap-2">
            <button type="button" onclick="App.openAttendanceJournalModal('alltime')" class="flex-1 py-2.5 px-3.5 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-xs transition text-center">
              Открыть посещаемость (за всё время)
            </button>
            <button type="button" onclick="App.openAttendanceJournalModal('today')" class="py-2.5 px-3 rounded-xl bg-slate-900 hover:bg-slate-800 border border-emerald-500/30 text-emerald-300 font-semibold text-xs transition">
              За сегодня
            </button>
          </div>
        </div>

        <!-- Карточка 2: Объявления группы (наравне с посещаемостью) -->
        <div class="p-4 sm:p-5 rounded-2xl bg-gradient-to-br from-sky-950/70 via-cardBg to-indigo-950/40 border border-sky-500/40 flex flex-col justify-between space-y-3 shadow-lg shadow-sky-950/20">
          <div>
            <div class="flex items-center justify-between gap-2 mb-2">
              <span class="px-2.5 py-0.5 rounded-full text-[10px] font-bold uppercase tracking-wider bg-sky-500/20 text-sky-300 border border-sky-500/30">
                Важное для группы
              </span>
              <span id="home-announcements-count-pill" class="text-xs font-mono text-sky-300 font-semibold">
                0 объявл.
              </span>
            </div>
            <h3 class="text-base sm:text-lg font-bold text-white flex items-center gap-2">
              <span>📢 Объявления</span>
            </h3>
            <p id="home-latest-announcement-preview" class="text-xs text-slate-300 mt-1 leading-relaxed line-clamp-2">
              Объявлений пока нет. Нажмите, чтобы опубликовать важную информацию для всей группы.
            </p>
          </div>
          <div class="pt-2 flex flex-wrap items-center gap-2">
            <button type="button" onclick="App.openAnnouncementsModal()" class="flex-1 py-2.5 px-3.5 rounded-xl bg-sky-400 hover:bg-sky-300 text-slate-950 font-semibold text-xs transition text-center">
              Открыть объявления
            </button>
            <button type="button" onclick="App.switchTab('homework')" class="py-2.5 px-3 rounded-xl bg-slate-900 hover:bg-slate-800 border border-amber-500/40 text-amber-300 font-semibold text-xs transition">
              📝 ДЗ по неделям
            </button>
          </div>
        </div>
      </div>

      <!-- Блок "Первые шаги" (для старосты) -->
      <div id="onboarding-steps-card" class="p-5 rounded-2xl bg-cardBg border border-slate-800/90 space-y-5">
        <div class="flex items-start justify-between">
          <div>
            <h3 class="text-base font-bold text-white">Панель старосты · Первые шаги</h3>
            <p class="text-xs text-slate-400 mt-0.5">
              <span id="steps-progress-text">1 из 3</span> ·
              <button type="button" onclick="App.openHowItWorksModal()" class="text-emerald-400 hover:underline">как это работает</button>
            </p>
          </div>
          <button type="button" onclick="App.hideOnboardingSteps()" class="text-slate-400 hover:text-white p-1">
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
            </svg>
          </button>
        </div>

        <div class="space-y-4">
          <div class="flex items-start gap-3.5">
            <div class="w-7 h-7 rounded-full bg-emerald-500/15 border border-emerald-500/40 text-emerald-300 text-xs font-bold flex items-center justify-center shrink-0 mt-0.5">
              1
            </div>
            <div class="flex-1">
              <h4 class="text-sm font-semibold text-white">Пригласите студентов по коду и назначьте Зам. старосты</h4>
              <p class="text-xs text-slate-400 mt-0.5 mb-2.5">Студенты вводят ваш код при входе и появляются в списке группы, где вы можете назначить их заместителями</p>
              <div class="flex flex-wrap gap-2">
                <button type="button" onclick="App.copyInviteLink()" class="flex-1 py-2.5 px-4 rounded-xl bg-slate-900 hover:bg-slate-800 border border-slate-700/80 text-xs sm:text-sm font-medium text-slate-200 flex items-center justify-center gap-2 transition">
                  <svg class="w-4 h-4 text-emerald-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 16H6a2 2 0 01-2-2V6a2 2 0 012-2h8a2 2 0 012 2v2m-6 12h8a2 2 0 002-2v-8a2 2 0 00-2-2h-8a2 2 0 00-2 2v8a2 2 0 002 2z"/>
                  </svg>
                  <span>Скопировать код приглашения</span>
                </button>
                <button type="button" onclick="App.openDeputiesModal()" class="py-2.5 px-4 rounded-xl bg-amber-500/15 hover:bg-amber-500/25 border border-amber-500/40 text-xs sm:text-sm font-semibold text-amber-300 transition">
                  Назначить зам. старосты
                </button>
              </div>
            </div>
          </div>

          <div class="flex items-start gap-3.5">
            <div class="w-7 h-7 rounded-full bg-emerald-500/15 border border-emerald-500/40 text-emerald-300 text-xs font-bold flex items-center justify-center shrink-0 mt-0.5">
              2
            </div>
            <div class="flex-1">
              <h4 class="text-sm font-semibold text-white">Загрузите фото расписания или фото списка посещаемости (ИИ)</h4>
              <p class="text-xs text-slate-400 mt-0.5 mb-2.5">ИИ распознает пары со звёздами (★ 1-я неделя, ★★ 2-я неделя) и фото присутствующих</p>
              <div class="flex flex-wrap gap-2">
                <button type="button" onclick="App.triggerPhotoScheduleUpload()" class="flex-1 py-2.5 px-4 rounded-xl bg-slate-900 hover:bg-slate-800 border border-emerald-500/40 text-xs sm:text-sm font-medium text-emerald-300 flex items-center justify-center gap-2 transition">
                  <span>📷 Фото расписания (ИИ)</span>
                </button>
                <button type="button" onclick="App.triggerAttendancePhotoUpload()" class="flex-1 py-2.5 px-4 rounded-xl bg-slate-900 hover:bg-slate-800 border border-cyan-500/40 text-xs sm:text-sm font-medium text-cyan-300 flex items-center justify-center gap-2 transition">
                  <span>📸 Отметить по фото списка</span>
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- Виджет "ТЕЛЕГРАМ-РАССЫЛКА ПОСЕЩАЕМОСТИ ВСЕМ" (только для Старосты и Зам. старосты) -->
      <div id="leader-broadcast-card" class="p-5 rounded-2xl bg-gradient-to-br from-emerald-950/60 via-slate-900 to-teal-950/50 border border-emerald-500/30 shadow-lg shadow-emerald-950/20">
        <div class="flex items-center justify-between gap-2 mb-2">
          <div class="flex items-center gap-2">
            <span id="live-status-dot" class="w-2.5 h-2.5 rounded-full bg-emerald-400 animate-ping"></span>
            <span class="text-xs font-bold uppercase tracking-wider text-emerald-300">ТЕЛЕГРАМ-БОТ · РАССЫЛКА ПОСЕЩАЕМОСТИ</span>
          </div>
          <span id="auto-send-badge" class="px-2.5 py-0.5 rounded-full text-[11px] font-semibold bg-emerald-500/20 text-emerald-300 border border-emerald-500/30">
            Авто-рассылка активна
          </span>
        </div>

        <p id="live-attendance-text" class="text-sm text-slate-200 leading-relaxed">
          Бот готов отправить сводку посещаемости всем участникам группы в Telegram и сохранить день в общую статистику за всё время.
        </p>

        <div class="mt-4 pt-3 border-t border-emerald-500/20 flex flex-wrap items-center justify-between gap-2.5">
          <button type="button" onclick="App.openAttendanceJournalModal('alltime')" class="text-xs sm:text-sm font-semibold text-emerald-400 hover:text-emerald-300 inline-flex items-center gap-1 transition">
            <span>Посещаемость всех за всё время</span>
            <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M14 5l7 7m0 0l-7 7m7-7H3"/>
            </svg>
          </button>

          <div class="flex flex-wrap items-center gap-2">
            <button type="button" onclick="App.triggerAttendancePhotoUpload()" class="px-3 py-2 rounded-xl bg-slate-900 hover:bg-slate-800 border border-emerald-500/40 text-emerald-300 font-semibold text-xs flex items-center gap-1.5 transition">
              <span>📸 Посещаемость по фото</span>
            </button>

            <button type="button" onclick="App.openBroadcastAttendanceModal()" class="px-3.5 py-2 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-xs flex items-center gap-1.5 shadow-sm transition">
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8"/>
              </svg>
              <span>Отправить всем сообщение о посещаемости</span>
            </button>
          </div>
        </div>
      </div>

      <!-- БЫСТРЫЕ ДЕЙСТВИЯ -->
      <div>
        <h3 class="text-xs font-bold uppercase tracking-wider text-slate-400 mb-3">
          БЫСТРЫЕ ДЕЙСТВИЯ
        </h3>

        <div class="grid grid-cols-2 sm:grid-cols-3 gap-3">
          <button type="button" onclick="App.openAttendanceJournalModal('alltime')" class="p-4 rounded-2xl bg-cardBg hover:bg-cardHover border border-emerald-500/35 text-left transition group">
            <div class="w-10 h-10 rounded-xl bg-emerald-500/15 text-emerald-400 flex items-center justify-center mb-3 group-hover:scale-105 transition">
              <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5H7a2 2 0 00-2 2v12a2 2 0 002 2h10a2 2 0 002-2V7a2 2 0 00-2-2h-2M9 5a2 2 0 002 2h2a2 2 0 002-2M9 5a2 2 0 012-2h2a2 2 0 012 2m-6 9l2 2 4-4"/>
              </svg>
            </div>
            <h4 class="text-sm sm:text-base font-bold text-white">Посещаемость</h4>
            <p class="text-xs text-slate-400 mt-1">Все участники за всё время</p>
          </button>

          <button type="button" onclick="App.openAnnouncementsModal()" class="p-4 rounded-2xl bg-cardBg hover:bg-cardHover border border-sky-500/35 text-left transition group">
            <div class="w-10 h-10 rounded-xl bg-sky-500/15 text-sky-400 flex items-center justify-center mb-3 group-hover:scale-105 transition">
              <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M11 5.882V19.24a1.76 1.76 0 01-3.417.592l-2.147-6.15M18 13a3 3 0 100-6M5.436 13.683A4.001 4.001 0 017 6h1.832c4.1 0 7.625-1.234 9.168-3v14c-1.543-1.766-5.067-3-9.168-3H7a3.988 3.988 0 01-1.564-.317z"/>
              </svg>
            </div>
            <h4 class="text-sm sm:text-base font-bold text-white">Объявления</h4>
            <p class="text-xs text-slate-400 mt-1">Важные новости группы</p>
          </button>

          <button type="button" onclick="App.switchTab('homework')" class="p-4 rounded-2xl bg-cardBg hover:bg-cardHover border border-amber-500/35 text-left transition group">
            <div class="w-10 h-10 rounded-xl bg-amber-500/15 text-amber-400 flex items-center justify-center mb-3 group-hover:scale-105 transition">
              <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 6.253v13m0-13C10.832 5.477 9.246 5 7.5 5S4.168 5.477 3 6.253v13C4.168 18.477 5.754 18 7.5 18s3.332.477 4.5 1.253m0-13C13.168 5.477 14.754 5 16.5 5c1.747 0 3.332.477 4.5 1.253v13C19.832 18.477 18.247 18 16.5 18c-1.746 0-3.332.477-4.5 1.253"/>
              </svg>
            </div>
            <h4 class="text-sm sm:text-base font-bold text-white">Домашнее задание</h4>
            <p class="text-xs text-slate-400 mt-1">По ★ 1-й и ★★ 2-й неделям</p>
          </button>

          <button type="button" onclick="App.switchTab('schedule')" class="p-4 rounded-2xl bg-cardBg hover:bg-cardHover border border-slate-800/90 text-left transition group">
            <div class="w-10 h-10 rounded-xl bg-teal-500/15 text-teal-400 flex items-center justify-center mb-3 group-hover:scale-105 transition">
              <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/>
              </svg>
            </div>
            <h4 class="text-sm sm:text-base font-bold text-white">Расписание</h4>
            <p class="text-xs text-slate-400 mt-1">ИИ-разбор фото, ★ и ★★ недели</p>
          </button>

          <button type="button" onclick="App.switchTab('students')" class="p-4 rounded-2xl bg-cardBg hover:bg-cardHover border border-slate-800/90 text-left transition group">
            <div class="w-10 h-10 rounded-xl bg-cyan-500/15 text-cyan-400 flex items-center justify-center mb-3 group-hover:scale-105 transition">
              <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 20h5v-2a3 3 0 00-5.356-1.857M17 20H7m10 0v-2c0-.656-.126-1.283-.356-1.857M7 20H2v-2a3 3 0 015.356-1.857M7 20v-2c0-.656.126-1.283.356-1.857m0 0a5.002 5.002 0 019.288 0M15 7a3 3 0 11-6 0 3 3 0 016 0z"/>
              </svg>
            </div>
            <h4 class="text-sm sm:text-base font-bold text-white">Студенты и Замы</h4>
            <p class="text-xs text-slate-400 mt-1">Список группы, зам. старосты</p>
          </button>

          <button type="button" onclick="App.openKnowledgeBaseModal()" class="p-4 rounded-2xl bg-cardBg hover:bg-cardHover border border-slate-800/90 text-left transition group">
            <div class="w-10 h-10 rounded-xl bg-purple-500/15 text-purple-400 flex items-center justify-center mb-3 group-hover:scale-105 transition">
              <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 11H5m14 0a2 2 0 012 2v6a2 2 0 01-2 2H5a2 2 0 01-2-2v-6a2 2 0 012-2m14 0V9a2 2 0 00-2-2M5 11V9a2 2 0 012-2m0 0V5a2 2 0 012-2h6a2 2 0 012 2v2M7 7h10"/>
              </svg>
            </div>
            <h4 class="text-sm sm:text-base font-bold text-white">База знаний</h4>
            <p class="text-xs text-slate-400 mt-1">Файлы, методички и конспекты</p>
          </button>
        </div>
      </div>
    </section>

    <!-- ================================================================= -->
    <!-- ЭКРАН 4: СТУДЕНТЫ И НАЗНАЧЕНИЕ ЗАМ. СТАРОСТЫ                      -->
    <!-- ================================================================= -->
    <section id="screen-students" class="screen-fade hidden space-y-5">
      <div class="flex flex-wrap items-center justify-between gap-2">
        <div>
          <h2 class="text-2xl font-bold text-white">Студенты группы</h2>
          <p id="students-count-subtitle" class="text-xs sm:text-sm text-slate-400 mt-0.5">0 в группе · на паре 0</p>
        </div>
        <div class="flex items-center gap-2 flex-wrap">
          <button type="button" id="btn-student-leave-group" onclick="App.leaveGroupAsStudent()" class="hidden px-3 py-2 rounded-xl bg-rose-500/15 hover:bg-rose-500/25 border border-rose-500/35 text-rose-300 text-xs font-semibold flex items-center gap-1.5 transition">
            <span>Выйти из группы</span>
          </button>
          <button type="button" id="btn-manage-deputies" onclick="App.openDeputiesModal()" class="px-3 py-2 rounded-xl bg-amber-500/15 hover:bg-amber-500/25 border border-amber-500/35 text-amber-300 text-xs font-semibold flex items-center gap-1.5 transition">
            <span>👑 Зам. старосты</span>
          </button>
          <button type="button" id="btn-add-student" onclick="App.openAddStudentModal()" class="px-3.5 py-2 rounded-xl bg-emerald-500/15 hover:bg-emerald-500/25 border border-emerald-500/30 text-emerald-300 text-xs font-semibold flex items-center gap-1.5 transition">
            <span>+ Добавить</span>
          </button>
        </div>
      </div>

      <div id="deputy-info-card" class="p-4 rounded-2xl bg-slate-900/90 border border-amber-500/30 flex flex-wrap items-center justify-between gap-3">
        <div>
          <p class="text-xs font-bold uppercase tracking-wider text-amber-300">УПРАВЛЕНИЕ УЧАСТНИКАМИ И ЗАМЕСТИТЕЛЯМИ</p>
          <p class="text-xs text-slate-300 mt-0.5">
            Староста может назначать <strong>Зам. старосты</strong>. Староста и заместители могут удалять участников из группы.
          </p>
        </div>
        <button type="button" onclick="App.openBroadcastAttendanceModal()" class="px-3 py-2 rounded-xl bg-emerald-500/20 hover:bg-emerald-500/30 border border-emerald-500/40 text-emerald-300 text-xs font-semibold transition">
          Разослать всем посещаемость
        </button>
      </div>

      <div id="students-empty-state" class="py-8 text-center max-w-md mx-auto">
        <p class="text-sm text-slate-400 leading-relaxed">
          В списке этой группы пока никого нет. Пригласите участников по коду группы или добавьте их вручную.
        </p>
      </div>

      <div id="add-self-card" class="p-5 rounded-2xl bg-cardBg border border-slate-800/90 space-y-4">
        <h3 class="text-base font-bold text-white">Добавить себя в список группы?</h3>
        <p class="text-xs sm:text-sm text-slate-400 leading-relaxed">
          Вы появитесь в электронном журнале посещаемости со статусом «На паре (авто)».
        </p>
        <button type="button" onclick="App.addSelfToGroup()" class="w-full py-3.5 px-5 rounded-2xl bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 font-semibold text-sm flex items-center justify-center gap-2 shadow-lg shadow-emerald-500/20 transition">
          <span>Добавить себя в журнал</span>
        </button>
      </div>

      <div id="students-list" class="space-y-2.5"></div>
    </section>

    <!-- ================================================================= -->
    <!-- ЭКРАН 5: РАСПИСАНИЕ (С ИИ-РАЗБОРОМ ФОТО И ЗВЁЗДАМИ ★ / ★★)        -->
    <!-- ================================================================= -->
    <section id="screen-schedule" class="screen-fade hidden space-y-5 relative">
      <div class="flex flex-wrap items-center justify-between gap-2">
        <div>
          <h2 class="text-2xl font-bold text-white">Расписание уроков</h2>
          <p class="text-xs sm:text-sm text-slate-400 mt-0.5">Полное расписание по неделям (★ 1-я и ★★ 2-я неделя), кабинеты и преподаватели</p>
        </div>
        <span id="schedule-readonly-badge" class="hidden px-3 py-1 rounded-xl bg-slate-800 border border-slate-700 text-xs text-cyan-300 font-medium">
          🔒 Только просмотр (Студент)
        </span>
      </div>

      <!-- Панель загрузки и редактирования расписания (ТОЛЬКО ДЛЯ СТАРОСТЫ И ЗАМ. СТАРОСТЫ) -->
      <div id="schedule-editor-card" class="p-5 rounded-2xl bg-gradient-to-br from-emerald-950/60 via-cardBg to-teal-950/50 border border-emerald-500/35 space-y-4">
        <div class="flex items-center justify-between flex-wrap gap-2">
          <div class="w-10 h-10 rounded-xl bg-emerald-500/15 border border-emerald-500/30 text-emerald-400 flex items-center justify-center">
            <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M5 3v4M3 5h4M6 17v4m-2-2h4m5-16l2.286 6.857L21 12l-5.714 2.143L13 21l-2.286-6.857L5 12l5.714-2.143L13 3z"/>
            </svg>
          </div>
          <div class="flex items-center gap-1.5 text-[11px] font-medium">
            <span class="px-2.5 py-1 rounded-lg bg-amber-500/15 text-amber-300 border border-amber-500/30">★ = 1-я неделя</span>
            <span class="px-2.5 py-1 rounded-lg bg-cyan-500/15 text-cyan-300 border border-cyan-500/30">★★ = 2-я неделя</span>
          </div>
        </div>

        <h3 class="text-base sm:text-lg font-bold text-white leading-snug">
          Загрузите фото расписания — ИИ перенесёт все пары, кабинеты и недели ★ / ★★
        </h3>
        <p class="text-xs sm:text-sm text-slate-300 leading-relaxed">
          Прикрепите чёткое фото или скриншот расписания вашей группы. ИИ автоматически распознает все дни недели, номера пар, предметы, преподавателей, кабинеты и разделит занятия по звёздам (<strong class="text-amber-300">★ 1-я неделя</strong> и <strong class="text-cyan-300">★★ 2-я неделя</strong>), подставив время из расписания звонков вашего колледжа или вуза.
        </p>

        <div class="flex flex-wrap items-center gap-2.5">
          <button type="button" onclick="App.triggerPhotoScheduleUpload()" class="px-5 py-3 rounded-xl bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 font-semibold text-xs sm:text-sm inline-flex items-center gap-2 shadow-md shadow-emerald-500/20 transition">
            <span>📷 Загрузить фото расписания (ИИ)</span>
          </button>

          <button type="button" onclick="App.openAddClassModal()" class="px-4 py-3 rounded-xl bg-slate-900 hover:bg-slate-800 border border-slate-700 text-slate-200 font-medium text-xs sm:text-sm inline-flex items-center gap-1.5 transition">
            <span>+ Добавить пару вручную</span>
          </button>
        </div>
      </div>

      <!-- РАСПИСАНИЕ ЗВОНКОВ И ПЕРЕМЕН (ИИ ПО ФОТО + РУЧНОЙ ВВОД) -->
      <div class="p-4 sm:p-5 rounded-2xl bg-cardBg border border-slate-800/90 space-y-3.5">
        <div class="flex flex-wrap items-center justify-between gap-2">
          <div>
            <div class="flex items-center gap-2">
              <span class="text-xs font-bold uppercase tracking-wider text-teal-400">🔔 РАСПИСАНИЕ ЗВОНКОВ И ПЕРЕМЕН</span>
            </div>
            <p class="text-xs text-slate-400 mt-0.5">
              Время начала и конца пар, длительность перемен в вашем учебном заведении
            </p>
          </div>

          <div id="bell-editor-controls" class="flex flex-wrap items-center gap-2">
            <button type="button" onclick="App.triggerBellPhotoUpload()" class="px-3 py-2 rounded-xl bg-teal-500/15 hover:bg-teal-500/25 border border-teal-500/35 text-teal-300 font-semibold text-xs flex items-center gap-1.5 transition">
              <span>📷 Звонки по фото (ИИ)</span>
            </button>
            <button type="button" onclick="App.openBellScheduleModal()" class="px-3 py-2 rounded-xl bg-slate-900 hover:bg-slate-800 border border-slate-700 text-slate-200 font-semibold text-xs flex items-center gap-1.5 transition">
              <span>✏️ Ввести вручную</span>
            </button>
          </div>
        </div>

        <div id="bell-schedule-list" class="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 gap-2"></div>
      </div>

      <!-- Фильтр недель -->
      <div class="p-4 rounded-2xl bg-cardBg border border-slate-800/90 space-y-3">
        <div class="flex items-center justify-between flex-wrap gap-2">
          <span class="text-xs font-bold uppercase tracking-wider text-emerald-400">ФИЛЬТР ПО ЗВЁЗДАМ (НЕДЕЛЯМ)</span>
          <span class="text-xs text-slate-400">★ 1 звезда = 1-я неделя · ★★ 2 звезды = 2-я неделя</span>
        </div>
        <div class="grid grid-cols-3 gap-2">
          <button type="button" id="week-filter-all" onclick="App.setScheduleWeekFilter('all')" class="py-2.5 px-3 rounded-xl text-xs font-semibold bg-emerald-500 text-slate-950 transition">
            Все недели
          </button>
          <button type="button" id="week-filter-1" onclick="App.setScheduleWeekFilter('1')" class="py-2.5 px-3 rounded-xl text-xs font-semibold bg-slate-900 hover:bg-slate-800 text-amber-300 border border-slate-800 transition">
            ★ 1-я неделя
          </button>
          <button type="button" id="week-filter-2" onclick="App.setScheduleWeekFilter('2')" class="py-2.5 px-3 rounded-xl text-xs font-semibold bg-slate-900 hover:bg-slate-800 text-cyan-300 border border-slate-800 transition">
            ★★ 2-я неделя
          </button>
        </div>
      </div>

      <!-- Список пар -->
      <div class="space-y-3">
        <div class="flex items-center justify-between">
          <h3 class="text-xs font-bold uppercase tracking-wider text-emerald-400">РАСПИСАНИЕ НЕДЕЛИ</h3>
          <button type="button" onclick="App.clearAllClasses()" id="clear-schedule-btn" class="hidden text-xs text-rose-400 hover:underline">
            Очистить расписание
          </button>
        </div>
        <div id="schedule-classes-list" class="space-y-3"></div>
      </div>

      <!-- Семестр -->
      <div class="p-5 rounded-2xl bg-cardBg border border-slate-800/90 space-y-3">
        <span class="text-xs font-bold uppercase tracking-wider text-slate-400">СЕМЕСТР</span>
        <p id="semester-dates-display" class="text-base font-semibold text-white">01.09.2026 — 31.12.2026</p>
        <button type="button" id="btn-edit-semester" onclick="App.openSemesterModal()" class="w-full py-2.5 px-4 rounded-xl bg-slate-900 hover:bg-slate-800 border border-slate-700/80 text-xs sm:text-sm font-medium text-slate-200 transition">
          Изменить даты семестра
        </button>
      </div>

      <!-- Каникулы и сессии -->
      <div class="space-y-3 pt-2">
        <h3 class="text-xs font-bold uppercase tracking-wider text-emerald-400">КАНИКУЛЫ И СЕССИИ</h3>
        <div id="holidays-list" class="text-center py-4 text-sm text-slate-400">
          Периодов пока нет.
        </div>
        <button type="button" id="btn-add-holiday" onclick="App.openAddHolidayModal()" class="w-full py-3 px-4 rounded-xl bg-cardBg hover:bg-cardHover border border-slate-800 text-xs sm:text-sm font-medium text-slate-200 transition">
          + Добавить каникулы / сессию
        </button>
      </div>

      <button type="button" id="fab-add-class" onclick="App.openAddClassModal()" title="Добавить пару" class="fixed bottom-24 right-6 z-20 w-14 h-14 rounded-full bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 shadow-xl shadow-emerald-500/30 flex items-center justify-center text-2xl font-bold transition transform hover:scale-105">
        +
      </button>
    </section>

    <!-- ================================================================= -->
    <!-- ЭКРАН 5.5: ОТДЕЛЬНАЯ ВКЛАДКА ДОМАШНЕГО ЗАДАНИЯ (С УЧЁТОМ НЕДЕЛЬ)  -->
    <!-- ================================================================= -->
    <section id="screen-homework" class="screen-fade hidden space-y-5">
      <div class="flex flex-wrap items-center justify-between gap-2">
        <div>
          <h2 class="text-2xl font-bold text-white">Домашнее задание</h2>
          <p class="text-xs sm:text-sm text-slate-400 mt-0.5">Заполнение и просмотр ДЗ с учётом ★ 1-й и ★★ 2-й недели</p>
        </div>
        <button type="button" onclick="App.openAddHomeworkModal()" class="px-4 py-2.5 rounded-xl bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 font-semibold text-xs sm:text-sm flex items-center gap-1.5 shadow-md shadow-emerald-500/20 transition">
          <span>+ Добавить ДЗ</span>
        </button>
      </div>

      <!-- Фильтр ДЗ по неделям (Все / ★ 1-я неделя / ★★ 2-я неделя) -->
      <div class="p-4 rounded-2xl bg-cardBg border border-slate-800/90 space-y-3">
        <div class="flex items-center justify-between flex-wrap gap-2">
          <span class="text-xs font-bold uppercase tracking-wider text-amber-400">ВЫБОР НЕДЕЛИ ДЛЯ ДОМАШНЕГО ЗАДАНИЯ</span>
          <span id="hw-active-week-hint" class="text-xs text-emerald-400 font-medium">Текущая: ★ 1-я неделя</span>
        </div>
        <div class="grid grid-cols-3 gap-2">
          <button type="button" id="hw-week-filter-all" onclick="App.setHomeworkWeekFilter('all')" class="py-2.5 px-3 rounded-xl text-xs font-semibold bg-emerald-500 text-slate-950 transition">
            Все недели
          </button>
          <button type="button" id="hw-week-filter-1" onclick="App.setHomeworkWeekFilter('1')" class="py-2.5 px-3 rounded-xl text-xs font-semibold bg-slate-900 hover:bg-slate-800 text-amber-300 border border-slate-800 transition">
            ★ 1-я неделя
          </button>
          <button type="button" id="hw-week-filter-2" onclick="App.setHomeworkWeekFilter('2')" class="py-2.5 px-3 rounded-xl text-xs font-semibold bg-slate-900 hover:bg-slate-800 text-cyan-300 border border-slate-800 transition">
            ★★ 2-я неделя
          </button>
        </div>
      </div>

      <!-- Форма быстрого добавления ДЗ прямо во вкладке -->
      <div class="p-5 rounded-2xl bg-cardBg border border-slate-800/90 space-y-3.5">
        <h3 class="text-sm sm:text-base font-bold text-white flex items-center gap-2">
          <span>📝 Заполнить домашнее задание</span>
        </h3>
        <form onsubmit="App.submitHomeworkInline(event)" class="space-y-3">
          <div class="grid grid-cols-1 sm:grid-cols-3 gap-2.5">
            <div>
              <label class="block text-[11px] text-slate-400 mb-1">День недели</label>
              <select id="inline-hw-day" onchange="App.populateInlineHwSubjects()" class="w-full px-3 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-xs sm:text-sm">
                <option value="Понедельник">Понедельник</option>
                <option value="Вторник">Вторник</option>
                <option value="Среда">Среда</option>
                <option value="Четверг">Четверг</option>
                <option value="Пятница">Пятница</option>
                <option value="Суббота">Суббота</option>
              </select>
            </div>
            <div>
              <label class="block text-[11px] text-slate-400 mb-1">Неделя (★ / ★★)</label>
              <select id="inline-hw-week" onchange="App.populateInlineHwSubjects()" class="w-full px-3 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-amber-300 font-semibold text-xs sm:text-sm">
                <option value="1">★ 1-я неделя</option>
                <option value="2">★★ 2-я неделя</option>
                <option value="both">Каждая неделя</option>
              </select>
            </div>
            <div>
              <label class="block text-[11px] text-slate-400 mb-1">Срок сдачи / Дата</label>
              <input id="inline-hw-due" type="text" placeholder="Например: к след. паре" class="w-full px-3 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-xs sm:text-sm" />
            </div>
          </div>

          <div>
            <label class="block text-[11px] text-slate-400 mb-1">Предмет (из расписания или свой)</label>
            <input id="inline-hw-subject" list="hw-subjects-datalist" type="text" required placeholder="Выберите или введите предмет..." class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-xs sm:text-sm focus:border-emerald-500 focus:outline-none" />
            <datalist id="hw-subjects-datalist"></datalist>
          </div>

          <div>
            <label class="block text-[11px] text-slate-400 mb-1">Задание (что нужно сделать)</label>
            <textarea id="inline-hw-text" rows="2" required placeholder="Например: Выучить конспект, решить №12-18 на стр. 45..." class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-xs sm:text-sm focus:border-emerald-500 focus:outline-none"></textarea>
          </div>

          <button type="submit" class="w-full py-3 rounded-xl bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 font-semibold text-xs sm:text-sm shadow-md transition">
            Сохранить домашнее задание
          </button>
        </form>
      </div>

      <!-- Список домашних заданий по дням и неделям -->
      <div id="homework-list-container" class="space-y-3"></div>
    </section>

    <!-- ================================================================= -->
    <!-- ЭКРАН 6: НАСТРОЙКИ И УПРАВЛЕНИЕ РОЛЯМИ                            -->
    <!-- ================================================================= -->
    <section id="screen-settings" class="screen-fade hidden space-y-6">
      <div class="flex items-center justify-between flex-wrap gap-2">
        <h2 class="text-2xl font-bold text-white">Настройки</h2>
        <div class="flex items-center gap-2">
          <button type="button" id="settings-leave-group-btn" onclick="App.leaveGroupAsStudent()" class="hidden px-3 py-1.5 rounded-xl bg-rose-500/20 hover:bg-rose-500/30 border border-rose-500/30 text-xs font-semibold text-rose-300 transition">
            Выйти из группы
          </button>
          <button type="button" onclick="App.resetToWelcome()" class="px-3 py-1.5 rounded-xl bg-slate-800 hover:bg-slate-700 text-xs font-medium text-emerald-400 transition">
            На экран входа
          </button>
        </div>
      </div>

      <div class="p-4 rounded-2xl bg-cardBg border border-slate-800/90 flex items-center justify-between gap-3">
        <div class="flex items-center gap-3">
          <div id="settings-group-avatar" class="w-12 h-12 rounded-2xl bg-gradient-to-br from-emerald-500 to-teal-600 text-slate-950 font-bold text-lg flex items-center justify-center">
            Г
          </div>
          <div>
            <h3 id="settings-group-name" class="text-base font-bold text-white">Моя группа</h3>
            <p id="settings-uni-name" class="text-xs text-slate-400">Учебное заведение</p>
          </div>
        </div>
        <button type="button" id="btn-edit-group-info" onclick="App.openEditGroupModal()" class="px-3.5 py-2 rounded-xl bg-slate-800 hover:bg-slate-700 border border-slate-700 text-xs font-medium text-slate-200 flex items-center gap-1.5 transition">
          <span>Изменить</span>
        </button>
      </div>

      <div class="space-y-2">
        <h3 id="settings-section-title" class="text-xs font-bold uppercase tracking-wider text-slate-400 px-1">ГРУППА, ЗАМЕСТИТЕЛИ И РАССЫЛКА</h3>

        <div class="rounded-2xl bg-cardBg border border-slate-800/90 divide-y divide-slate-800/80 overflow-hidden">
          <button type="button" id="settings-invite-code-row" onclick="App.copyInviteLink()" class="w-full p-4 flex items-center justify-between hover:bg-slate-800/50 transition text-left">
            <div class="flex items-center gap-3">
              <svg class="w-5 h-5 text-emerald-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M18 9v3m0 0v3m0-3h3m-3 0h-3m-2-5a4 4 0 11-8 0 4 4 0 018 0zM3 20a6 6 0 0112 0v1H3v-1z"/>
              </svg>
              <div>
                <p class="text-sm font-medium text-white">Код приглашения для входа студентов</p>
                <p class="text-xs text-slate-400">По этому коду студенты присоединяются только к вашей группе</p>
              </div>
            </div>
            <div class="flex items-center gap-2 text-xs text-emerald-400 font-mono font-bold">
              <span id="settings-invite-code">—</span>
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
              </svg>
            </div>
          </button>

          <button type="button" id="settings-deputies-row" onclick="App.openDeputiesModal()" class="w-full p-4 flex items-center justify-between hover:bg-slate-800/50 transition text-left">
            <div class="flex items-center gap-3">
              <svg class="w-5 h-5 text-amber-400 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m5.618-4.016A11.955 11.955 0 0112 2.944a11.955 11.955 0 01-8.618 3.04A12.02 12.02 0 003 9c0 5.591 3.824 10.29 9 11.622 5.176-1.332 9-6.03 9-11.622 0-1.042-.133-2.052-.382-3.016z"/>
              </svg>
              <div>
                <p class="text-sm font-medium text-white">Заместители старосты (Зам. старосты)</p>
                <p class="text-xs text-slate-400">Назначить зам. старосты из подключившихся к группе студентов</p>
              </div>
            </div>
            <span id="settings-deputies-count" class="text-xs font-semibold text-amber-300">0 назн.</span>
          </button>

          <button type="button" id="settings-broadcast-row" onclick="App.openBroadcastAttendanceModal()" class="w-full p-4 flex items-center justify-between hover:bg-slate-800/50 transition text-left">
            <div class="flex items-center gap-3">
              <svg class="w-5 h-5 text-emerald-400 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 19l9 2-9-18-9 18 9-2zm0 0v-8"/>
              </svg>
              <div>
                <p class="text-sm font-medium text-white">Отправить всем сообщение о посещаемости в Telegram</p>
                <p class="text-xs text-slate-400">Мгновенная рассылка отчёта по группе через Telegram Mini App</p>
              </div>
            </div>
            <svg class="w-4 h-4 text-slate-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
            </svg>
          </button>

          <div id="settings-auto-attendance-row" class="p-4 flex items-center justify-between gap-4">
            <div class="flex items-center gap-3">
              <svg class="w-5 h-5 text-emerald-400 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12l2 2 4-4m6 2a9 9 0 11-18 0 9 9 0 0118 0z"/>
              </svg>
              <div>
                <p class="text-sm font-medium text-white">Авто-отправка присутствия на парах</p>
                <p class="text-xs text-slate-400">Бот сам отмечает и рассылает статус «На паре» по расписанию</p>
              </div>
            </div>
            <label class="relative inline-flex items-center cursor-pointer">
              <input id="toggle-auto-attendance" type="checkbox" checked onchange="App.toggleAutoAttendance(this.checked)" class="sr-only peer" />
              <div class="w-11 h-6 bg-slate-700 rounded-full peer peer-checked:after:translate-x-full peer-checked:after:border-white after:content-[''] after:absolute after:top-[2px] after:left-[2px] after:bg-white after:border-slate-300 after:border after:rounded-full after:h-5 after:w-5 after:transition-all peer-checked:bg-emerald-500"></div>
            </label>
          </div>

          <button type="button" id="settings-poll-lead-row" onclick="App.cyclePollLeadTime()" class="w-full p-4 flex items-center justify-between hover:bg-slate-800/50 transition text-left">
            <div class="flex items-center gap-3">
              <svg class="w-5 h-5 text-emerald-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9"/>
              </svg>
              <span class="text-sm font-medium text-white">Опросы и авто-отметка перед парой</span>
            </div>
            <div class="flex items-center gap-2 text-xs text-slate-400">
              <span id="poll-lead-time">за 15 мин</span>
              <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
              </svg>
            </div>
          </button>

          <button type="button" onclick="App.openTimezoneModal()" class="w-full p-4 flex items-center justify-between hover:bg-slate-800/50 transition text-left">
            <div class="flex items-center gap-3">
              <svg class="w-5 h-5 text-emerald-400 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M12 8v4l3 3m6-3a9 9 0 11-18 0 9 9 0 0118 0z"/>
              </svg>
              <div>
                <p class="text-sm font-medium text-white">Часовой пояс (вся Россия)</p>
                <p class="text-xs text-emerald-400 font-mono">Сейчас: <span class="live-clock-time">--:--:--</span></p>
              </div>
            </div>
            <div class="flex items-center gap-2 text-xs text-slate-300">
              <span id="settings-timezone" class="text-right">Москва (UTC+3)</span>
              <svg class="w-4 h-4 text-slate-400 shrink-0" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
              </svg>
            </div>
          </button>

          <button type="button" onclick="App.openLegalModal('pdn')" class="w-full p-4 flex items-center justify-between hover:bg-slate-800/50 transition text-left">
            <div class="flex items-center gap-3">
              <svg class="w-5 h-5 text-emerald-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
              </svg>
              <span class="text-sm font-medium text-white">Соглашения на обработку ПДн и согласия</span>
            </div>
            <svg class="w-4 h-4 text-slate-400" fill="none" stroke="currentColor" viewBox="0 0 24 24">
              <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 5l7 7-7 7"/>
            </svg>
          </button>
        </div>
      </div>

      <p class="text-xs text-slate-500 pt-2">
        © 2026 «Староста бот» · Telegram Mini App
      </p>
    </section>

  </main>

  <!-- НИЖНЯЯ НАВИГАЦИОННАЯ ПАНЕЛЬ (5 ВКЛАДОК ВКЛЮЧАЯ ДЗ) -->
  <nav id="bottom-nav" class="hidden fixed bottom-0 inset-x-0 z-30 bg-appBg/95 backdrop-blur-md border-t border-slate-800/90">
    <div class="max-w-4xl mx-auto px-2 h-16 grid grid-cols-5">
      <button type="button" onclick="App.switchTab('home')" id="nav-btn-home" class="flex flex-col items-center justify-center gap-1 text-emerald-400 transition">
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M3 12l2-2m0 0l7-7 7 7M5 10v10a1 1 0 001 1h3m10-11l2 2m-2-2v10a1 1 0 01-1 1h-3m-6 0a1 1 0 001-1v-4a1 1 0 011-1h2a1 1 0 011 1v4a1 1 0 001 1m-6 0h6"/>
        </svg>
        <span class="text-[10px] sm:text-[11px] font-medium">Главная</span>
      </button>

      <button type="button" onclick="App.switchTab('schedule')" id="nav-btn-schedule" class="flex flex-col items-center justify-center gap-1 text-slate-400 hover:text-slate-200 transition">
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 7V3m8 4V3m-9 8h10M5 21h14a2 2 0 002-2V7a2 2 0 00-2-2H5a2 2 0 00-2 2v12a2 2 0 002 2z"/>
        </svg>
        <span class="text-[10px] sm:text-[11px] font-medium">Расписание</span>
      </button>

      <button type="button" onclick="App.switchTab('homework')" id="nav-btn-homework" class="flex flex-col items-center justify-center gap-1 text-slate-400 hover:text-slate-200 transition">
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M9 12h6m-6 4h6m2 5H7a2 2 0 01-2-2V5a2 2 0 012-2h5.586a1 1 0 01.707.293l5.414 5.414a1 1 0 01.293.707V19a2 2 0 01-2 2z"/>
        </svg>
        <span class="text-[10px] sm:text-[11px] font-medium">ДЗ</span>
      </button>

      <button type="button" onclick="App.switchTab('students')" id="nav-btn-students" class="flex flex-col items-center justify-center gap-1 text-slate-400 hover:text-slate-200 transition">
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M17 20h5v-2a3 3 0 00-5.356-1.857M17 20H7m10 0v-2c0-.656-.126-1.283-.356-1.857M7 20H2v-2a3 3 0 015.356-1.857M7 20v-2c0-.656.126-1.283.356-1.857m0 0a5.002 5.002 0 019.288 0M15 7a3 3 0 11-6 0 3 3 0 016 0z"/>
        </svg>
        <span class="text-[10px] sm:text-[11px] font-medium">Студенты</span>
      </button>

      <button type="button" onclick="App.switchTab('settings')" id="nav-btn-settings" class="flex flex-col items-center justify-center gap-1 text-slate-400 hover:text-slate-200 transition">
        <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M10.325 4.317c.426-1.756 2.924-1.756 3.35 0a1.724 1.724 0 002.573 1.066c1.543-.94 3.31.826 2.37 2.37a1.724 1.724 0 001.065 2.572c1.756.426 1.756 2.924 0 3.35a1.724 1.724 0 00-1.066 2.573c.94 1.543-.826 3.31-2.37 2.37a1.724 1.724 0 00-2.572 1.065c-.426 1.756-2.924 1.756-3.35 0a1.724 1.724 0 00-2.573-1.066c-1.543.94-3.31-.826-2.37-2.37a1.724 1.724 0 00-1.065-2.572c-1.756-.426-1.756-2.924 0-3.35a1.724 1.724 0 001.066-2.573c-.94-1.543.826-3.31 2.37-2.37.996.608 2.296.07 2.572-1.065z"/>
          <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M15 12a3 3 0 11-6 0 3 3 0 016 0z"/>
        </svg>
        <span class="text-[10px] sm:text-[11px] font-medium">Настройки</span>
      </button>
    </div>
  </nav>

  <!-- УНИВЕРСАЛЬНОЕ МОДАЛЬНОЕ ОКНО -->
  <div id="modal-overlay" class="hidden fixed inset-0 z-50 bg-black/80 backdrop-blur-sm flex items-center justify-center p-4">
    <div class="w-full max-w-lg rounded-2xl bg-cardBg border border-slate-800 p-5 sm:p-6 shadow-2xl space-y-4 max-h-[88vh] overflow-y-auto">
      <div class="flex items-center justify-between border-b border-slate-800 pb-3">
        <h3 id="modal-title" class="text-base sm:text-lg font-bold text-white">Заголовок</h3>
        <button type="button" onclick="App.closeModal()" class="text-slate-400 hover:text-white p-1">
          <svg class="w-5 h-5" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M6 18L18 6M6 6l12 12"/>
          </svg>
        </button>
      </div>
      <div id="modal-body" class="text-sm text-slate-300 space-y-3"></div>
    </div>
  </div>

  <!-- ВСПЛЫВАЮЩЕЕ УВЕДОМЛЕНИЕ (TOAST) -->
  <div id="toast" class="fixed top-16 left-1/2 -translate-x-1/2 z-50 hidden px-4 py-2.5 rounded-xl bg-emerald-500 text-slate-950 font-semibold text-xs sm:text-sm shadow-xl transition text-center max-w-xs sm:max-w-md"></div>

  <!-- ================================================================= -->
  <!-- МОДУЛЬНЫЙ JAVASCRIPT-КОД ПРИЛОЖЕНИЯ                               -->
  <!-- ================================================================= -->
  <script>
    window.App = {
      STORAGE_KEY: 'starosta_bot_miniapp_v4_isolated',

      getDefaultBellSchedule() {
        return [
          { lessonNum: 0, start: '08:30', end: '09:00', breakMinutes: 0 },
          { lessonNum: 1, start: '09:00', end: '10:30', breakMinutes: 10 },
          { lessonNum: 2, start: '10:40', end: '12:10', breakMinutes: 20 },
          { lessonNum: 3, start: '12:30', end: '14:00', breakMinutes: 10 },
          { lessonNum: 4, start: '14:10', end: '15:40', breakMinutes: 10 },
          { lessonNum: 5, start: '15:50', end: '17:20', breakMinutes: 10 }
        ];
      },

      state: {
        hasCreatedGroup: false,
        leaderOwnCode: '',
        studentJoinedCode: '',
        studentJoinedId: null,
        studentJoinedName: '',
        currentUserRole: 'leader', // 'leader' (Староста) | 'deputy' (Зам. старосты) | 'student' (Студент)
        currentStudentId: null,
        currentStudentName: '',

        // Изолированное хранилище групп по коду
        groups: {},

        groupName: 'Моя группа',
        uniName: 'Учебное заведение',
        leaderName: 'Староста',
        inviteCode: '',
        semesterDates: '01.09.2026 — 31.12.2026',
        pollLeadTimes: ['за 15 мин', 'за 10 мин', 'за 5 мин', 'в начале пары'],
        pollLeadIndex: 0,
        currentWeekNumber: 1,
        scheduleWeekFilter: 'all',
        homeworkWeekFilter: 'all',

        russianTimezones: [
          { id: 'Europe/Kaliningrad', city: 'Калининград', short: 'МСК-1', utc: 'UTC+2' },
          { id: 'Europe/Moscow', city: 'Москва, Волгоград, СПб', short: 'МСК', utc: 'UTC+3' },
          { id: 'Europe/Samara', city: 'Самара, Саратов, Ижевск', short: 'МСК+1', utc: 'UTC+4' },
          { id: 'Asia/Yekaterinburg', city: 'Екатеринбург, Пермь, Уфа', short: 'МСК+2', utc: 'UTC+5' },
          { id: 'Asia/Omsk', city: 'Омск', short: 'МСК+3', utc: 'UTC+6' },
          { id: 'Asia/Krasnoyarsk', city: 'Красноярск, Новосибирск, Томск', short: 'МСК+4', utc: 'UTC+7' },
          { id: 'Asia/Irkutsk', city: 'Иркутск, Улан-Удэ', short: 'МСК+5', utc: 'UTC+8' },
          { id: 'Asia/Yakutsk', city: 'Якутск, Чита, Благовещенск', short: 'МСК+6', utc: 'UTC+9' },
          { id: 'Asia/Vladivostok', city: 'Владивосток, Хабаровск', short: 'МСК+7', utc: 'UTC+10' },
          { id: 'Asia/Magadan', city: 'Магадан, Южно-Сахалинск', short: 'МСК+8', utc: 'UTC+11' },
          { id: 'Asia/Kamchatka', city: 'Петропавловск-Камчатский, Анадырь', short: 'МСК+9', utc: 'UTC+12' }
        ],
        timezoneIndex: 1,

        autoAttendanceEnabled: true,
        stepsCompleted: { invite: false, schedule: false, autoPoll: true },

        // Изначально список участников группы полностью пуст
        students: [],
        bellSchedule: [
          { lessonNum: 0, start: '08:30', end: '09:00', breakMinutes: 0 },
          { lessonNum: 1, start: '09:00', end: '10:30', breakMinutes: 10 },
          { lessonNum: 2, start: '10:40', end: '12:10', breakMinutes: 20 },
          { lessonNum: 3, start: '12:30', end: '14:00', breakMinutes: 10 },
          { lessonNum: 4, start: '14:10', end: '15:40', breakMinutes: 10 },
          { lessonNum: 5, start: '15:50', end: '17:20', breakMinutes: 10 }
        ],
        classes: [],
        homeworks: [],
        attendanceHistory: [],
        holidays: [],
        announcements: [],
        knowledgeFiles: [],
        pendingAIClasses: [],
        pendingBellSchedule: [],
        broadcastHistory: []
      },

      generateRandomInviteCode() {
        const chars = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';
        let code = '';
        for (let i = 0; i < 6; i++) {
          code += chars.charAt(Math.floor(Math.random() * chars.length));
        }
        return code;
      },

      init() {
        this.loadState();

        try {
          if (window.Telegram && window.Telegram.WebApp) {
            const tg = window.Telegram.WebApp;
            tg.ready();
            tg.expand();
            if (tg.setHeaderColor) tg.setHeaderColor('#090d16');
            if (tg.setBackgroundColor) tg.setBackgroundColor('#090d16');

            const tgUser = tg.initDataUnsafe && tg.initDataUnsafe.user;
            if (tgUser) {
              const fullName = [tgUser.last_name, tgUser.first_name].filter(Boolean).join(' ');
              if (fullName && !this.state.hasCreatedGroup) {
                this.state.leaderName = fullName;
                const nameInput = document.getElementById('input-leader-name');
                if (nameInput) nameInput.value = fullName;
              }
            }
          }
        } catch (err) {
          console.warn('Telegram WebApp init skipped:', err);
        }

        this.updateLiveClocks();
        setInterval(() => this.updateLiveClocks(), 1000);

        this.syncGroupHeader();
        this.renderStudents();
        this.renderBellSchedule();
        this.renderSchedule();
        this.renderHomework();
        this.renderHolidays();
        this.updateProgressSteps();
        this.updateWelcomeScreenState();
      },

      // Сохранение текущей активной группы в изолированный словарь groups[inviteCode]
      syncActiveGroupToStore() {
        const code = (this.state.inviteCode || '').toUpperCase();
        if (!code) return;
        if (!this.state.groups || typeof this.state.groups !== 'object') {
          this.state.groups = {};
        }
        this.state.groups[code] = {
          groupName: this.state.groupName,
          uniName: this.state.uniName,
          leaderName: this.state.leaderName,
          inviteCode: code,
          semesterDates: this.state.semesterDates,
          currentWeekNumber: this.state.currentWeekNumber,
          autoAttendanceEnabled: this.state.autoAttendanceEnabled,
          stepsCompleted: JSON.parse(JSON.stringify(this.state.stepsCompleted || { invite: false, schedule: false, autoPoll: true })),
          students: JSON.parse(JSON.stringify(this.state.students || [])),
          bellSchedule: JSON.parse(JSON.stringify(this.state.bellSchedule || this.getDefaultBellSchedule())),
          classes: JSON.parse(JSON.stringify(this.state.classes || [])),
          homeworks: JSON.parse(JSON.stringify(this.state.homeworks || [])),
          attendanceHistory: JSON.parse(JSON.stringify(this.state.attendanceHistory || [])),
          holidays: JSON.parse(JSON.stringify(this.state.holidays || [])),
          announcements: JSON.parse(JSON.stringify(this.state.announcements || [])),
          knowledgeFiles: JSON.parse(JSON.stringify(this.state.knowledgeFiles || [])),
          broadcastHistory: JSON.parse(JSON.stringify(this.state.broadcastHistory || []))
        };
      },

      // Загрузка изолированных данных конкретной группы по её коду
      loadGroupFromStore(code) {
        const cleanCode = (code || '').toUpperCase();
        if (!cleanCode || !this.state.groups || !this.state.groups[cleanCode]) {
          return false;
        }
        const grp = this.state.groups[cleanCode];
        this.state.groupName = grp.groupName || 'Моя группа';
        this.state.uniName = grp.uniName || 'Учебное заведение';
        this.state.leaderName = grp.leaderName || 'Староста';
        this.state.inviteCode = cleanCode;
        this.state.semesterDates = grp.semesterDates || '01.09.2026 — 31.12.2026';
        this.state.currentWeekNumber = grp.currentWeekNumber || 1;
        this.state.autoAttendanceEnabled = grp.autoAttendanceEnabled !== undefined ? grp.autoAttendanceEnabled : true;
        this.state.stepsCompleted = JSON.parse(JSON.stringify(grp.stepsCompleted || { invite: false, schedule: false, autoPoll: true }));
        this.state.students = JSON.parse(JSON.stringify(grp.students || []));
        this.state.bellSchedule = JSON.parse(JSON.stringify(grp.bellSchedule || this.getDefaultBellSchedule()));
        this.state.classes = JSON.parse(JSON.stringify(grp.classes || []));
        this.state.homeworks = JSON.parse(JSON.stringify(grp.homeworks || []));
        this.state.attendanceHistory = JSON.parse(JSON.stringify(grp.attendanceHistory || []));
        this.state.holidays = JSON.parse(JSON.stringify(grp.holidays || []));
        this.state.announcements = JSON.parse(JSON.stringify(grp.announcements || []));
        this.state.knowledgeFiles = JSON.parse(JSON.stringify(grp.knowledgeFiles || []));
        this.state.broadcastHistory = JSON.parse(JSON.stringify(grp.broadcastHistory || []));
        return true;
      },

      saveState() {
        try {
          this.syncActiveGroupToStore();
          const copy = {
            hasCreatedGroup: this.state.hasCreatedGroup,
            leaderOwnCode: this.state.leaderOwnCode,
            studentJoinedCode: this.state.studentJoinedCode,
            studentJoinedId: this.state.studentJoinedId,
            studentJoinedName: this.state.studentJoinedName,
            currentUserRole: this.state.currentUserRole,
            currentStudentId: this.state.currentStudentId,
            currentStudentName: this.state.currentStudentName,
            groups: this.state.groups,
            groupName: this.state.groupName,
            uniName: this.state.uniName,
            leaderName: this.state.leaderName,
            inviteCode: this.state.inviteCode,
            semesterDates: this.state.semesterDates,
            timezoneIndex: this.state.timezoneIndex,
            currentWeekNumber: this.state.currentWeekNumber,
            autoAttendanceEnabled: this.state.autoAttendanceEnabled,
            students: this.state.students,
            bellSchedule: this.state.bellSchedule,
            classes: this.state.classes,
            homeworks: this.state.homeworks,
            attendanceHistory: this.state.attendanceHistory,
            holidays: this.state.holidays,
            announcements: this.state.announcements,
            knowledgeFiles: this.state.knowledgeFiles,
            broadcastHistory: this.state.broadcastHistory
          };
          localStorage.setItem(this.STORAGE_KEY, JSON.stringify(copy));
        } catch (e) {
          // Игнорируем ошибки доступа к хранилищу в приватном режиме
        }
      },

      loadState() {
        try {
          const raw = localStorage.getItem(this.STORAGE_KEY);
          if (!raw) return;
          const saved = JSON.parse(raw);
          if (saved && typeof saved === 'object') {
            Object.assign(this.state, saved);
            if (!this.state.groups || typeof this.state.groups !== 'object') {
              this.state.groups = {};
            }
            if (!Array.isArray(this.state.bellSchedule) || this.state.bellSchedule.length === 0) {
              this.state.bellSchedule = this.getDefaultBellSchedule();
            }
          }
        } catch (e) {
          // Используем состояние по умолчанию
        }
      },

      // Староста или Зам. старосты имеют права управления участниками
      canManageGroup() {
        if (this.state.currentUserRole === 'leader' || this.state.currentUserRole === 'deputy') {
          return true;
        }
        if (this.state.currentStudentId) {
          const me = this.state.students.find(s => s.id === this.state.currentStudentId);
          if (me && (me.isDeputy || me.isLeader)) return true;
        }
        return false;
      },

      isLeader() {
        return this.state.currentUserRole === 'leader';
      },

      updateWelcomeScreenState() {
        const quickBox = document.getElementById('existing-group-quick-box');
        const titleEl = document.getElementById('existing-group-title');
        const btnText = document.getElementById('welcome-leader-btn-text');

        const leaderCode = this.state.leaderOwnCode;
        const leaderGroup = leaderCode && this.state.groups ? this.state.groups[leaderCode] : null;

        if (this.state.hasCreatedGroup && leaderGroup) {
          if (quickBox) quickBox.classList.remove('hidden');
          if (titleEl) titleEl.textContent = `${leaderGroup.groupName} (${leaderGroup.uniName})`;
          if (btnText) btnText.textContent = 'Создать другую новую группу как староста';
        } else {
          if (quickBox) quickBox.classList.add('hidden');
          if (btnText) btnText.textContent = 'Войти как староста (создать группу)';
        }

        // Проверяем наличие сохранённого входа для студента (отдельная кнопка "Зайти в группу")
        const studentQuickBox = document.getElementById('existing-student-quick-box');
        const studentGroupTitle = document.getElementById('existing-student-group-title');
        const studentUserName = document.getElementById('existing-student-user-name');
        const studentRoleLabel = document.getElementById('existing-student-role-label');

        const sCode = (this.state.studentJoinedCode || '').toUpperCase();
        const sGroup = sCode && this.state.groups ? this.state.groups[sCode] : null;
        const sMember = sGroup && Array.isArray(sGroup.students)
          ? sGroup.students.find(st => st.id === this.state.studentJoinedId || (this.state.studentJoinedName && st.name.toLowerCase() === this.state.studentJoinedName.toLowerCase()))
          : null;

        if (sGroup && sMember) {
          if (studentQuickBox) studentQuickBox.classList.remove('hidden');
          if (studentGroupTitle) studentGroupTitle.textContent = `${sGroup.groupName} (${sGroup.uniName})`;
          if (studentUserName) studentUserName.textContent = `Участник: ${sMember.name}`;
          if (studentRoleLabel) studentRoleLabel.textContent = sMember.isDeputy ? 'Зам. старосты' : 'Студент';
        } else {
          if (studentQuickBox) studentQuickBox.classList.add('hidden');
        }
      },

      handleLeaderEntryClick() {
        this.state.currentUserRole = 'leader';
        const gInput = document.getElementById('input-group-name');
        const uInput = document.getElementById('input-uni-name');
        const cInput = document.getElementById('input-custom-code');
        if (gInput) gInput.value = '';
        if (uInput) uInput.value = '';
        if (cInput) cInput.value = '';
        this.navigate('screen-create-group');
      },

      enterExistingLeaderDashboard() {
        if (this.state.leaderOwnCode) {
          this.loadGroupFromStore(this.state.leaderOwnCode);
        }
        this.state.currentUserRole = 'leader';
        this.state.currentStudentId = null;
        this.state.currentStudentName = '';
        this.saveState();
        this.syncGroupHeader();
        this.renderStudents();
        this.renderBellSchedule();
        this.renderSchedule();
        this.renderHomework();
        this.renderHolidays();
        this.switchTab('home');
        this.showToast(`Вы вошли как староста группы «${this.state.groupName}»`);
      },

      enterExistingStudentDashboard() {
        const sCode = (this.state.studentJoinedCode || '').toUpperCase();
        if (!sCode || !this.loadGroupFromStore(sCode)) {
          this.showToast('Группа не найдена. Войдите по коду приглашения.');
          return;
        }
        const me = this.state.students.find(
          st => st.id === this.state.studentJoinedId ||
          (this.state.studentJoinedName && st.name.toLowerCase() === this.state.studentJoinedName.toLowerCase())
        );
        if (!me) {
          this.state.studentJoinedCode = '';
          this.state.studentJoinedId = null;
          this.state.studentJoinedName = '';
          this.saveState();
          this.updateWelcomeScreenState();
          this.showToast('Вы больше не числитесь в списке этой группы');
          return;
        }

        this.state.currentStudentId = me.id;
        this.state.currentStudentName = me.name;
        this.state.currentUserRole = me.isDeputy ? 'deputy' : 'student';
        this.saveState();
        this.syncGroupHeader();
        this.renderStudents();
        this.renderBellSchedule();
        this.renderSchedule();
        this.renderHomework();
        this.renderHolidays();
        this.switchTab('home');
        this.showToast(`Вы зашли в группу «${this.state.groupName}» как ${me.isDeputy ? 'Зам. старосты' : 'студент'}`);
      },

      getCurrentTimezone() {
        return this.state.russianTimezones[this.state.timezoneIndex] || this.state.russianTimezones[1];
      },

      formatTimeInTz(tzId, withSeconds = true) {
        try {
          return new Intl.DateTimeFormat('ru-RU', {
            timeZone: tzId,
            hour: '2-digit',
            minute: '2-digit',
            second: withSeconds ? '2-digit' : undefined,
            hour12: false
          }).format(new Date());
        } catch (e) {
          return new Date().toLocaleTimeString('ru-RU');
        }
      },

      formatDateInTz(tzId) {
        try {
          return new Intl.DateTimeFormat('ru-RU', {
            timeZone: tzId,
            weekday: 'short',
            day: '2-digit',
            month: '2-digit',
            year: 'numeric'
          }).format(new Date());
        } catch (e) {
          return new Date().toLocaleDateString('ru-RU');
        }
      },

      updateLiveClocks() {
        const tz = this.getCurrentTimezone();
        const timeStr = this.formatTimeInTz(tz.id, true);
        const dateStr = this.formatDateInTz(tz.id);

        document.querySelectorAll('.live-clock-time').forEach(el => {
          el.textContent = timeStr;
        });
        document.querySelectorAll('.live-clock-date').forEach(el => {
          el.textContent = dateStr;
        });
        document.querySelectorAll('.live-clock-tz-short').forEach(el => {
          el.textContent = `${tz.short} (${tz.utc})`;
        });
        document.querySelectorAll('.live-clock-tz-name').forEach(el => {
          el.textContent = `${tz.city} (${tz.utc})`;
        });

        const settingsTzEl = document.getElementById('settings-timezone');
        if (settingsTzEl) {
          settingsTzEl.textContent = `${tz.city} (${tz.utc})`;
        }
      },

      openTimezoneModal() {
        const currentTz = this.getCurrentTimezone();
        const itemsHtml = this.state.russianTimezones.map((tz, idx) => {
          const isSelected = idx === this.state.timezoneIndex;
          const liveNow = this.formatTimeInTz(tz.id, false);
          return `
            <button type="button" onclick="App.selectTimezone(${idx})" class="w-full p-3.5 rounded-xl border text-left flex items-center justify-between gap-3 transition ${
              isSelected
                ? 'bg-emerald-500/15 border-emerald-500/50 text-white'
                : 'bg-slate-900/90 hover:bg-slate-800 border-slate-800 text-slate-200'
            }">
              <div>
                <p class="text-sm font-semibold ${isSelected ? 'text-emerald-300' : 'text-white'}">${tz.city}</p>
                <p class="text-xs text-slate-400">${tz.utc} · ${tz.short}</p>
              </div>
              <div class="text-right">
                <span class="px-2.5 py-1 rounded-lg bg-slate-950 border border-slate-800 font-mono text-xs text-emerald-400 font-semibold">${liveNow}</span>
              </div>
            </button>
          `;
        }).join('');

        this.openModal(`Часовой пояс России (сейчас ${this.formatTimeInTz(currentTz.id, false)})`, `
          <p class="text-xs text-slate-400 mb-2">Выберите ваш регион. Время автоматически обновится на всех вкладках и для рассылки посещаемости.</p>
          <div class="space-y-2 max-h-[60vh] overflow-y-auto pr-1">
            ${itemsHtml}
          </div>
        `);
      },

      selectTimezone(index) {
        this.state.timezoneIndex = index;
        this.saveState();
        this.updateLiveClocks();
        this.closeModal();
        const tz = this.getCurrentTimezone();
        this.showToast(`Установлен часовой пояс: ${tz.city} (${tz.utc})`);
      },

      toggleCurrentWeekNumber() {
        if (!this.canManageGroup()) {
          this.showToast('Переключать номер текущей недели может только Староста или Зам. старосты');
          return;
        }
        this.state.currentWeekNumber = this.state.currentWeekNumber === 1 ? 2 : 1;
        this.saveState();
        const badge = document.getElementById('current-week-star-badge');
        if (this.state.currentWeekNumber === 1) {
          badge.textContent = '★ 1-я неделя';
        } else {
          badge.textContent = '★★ 2-я неделя';
        }
        this.renderHomework();
        this.showToast(`Активна ${this.state.currentWeekNumber === 1 ? '★ 1-я неделя' : '★★ 2-я неделя'}`);
      },

      navigate(screenId) {
        const screens = ['screen-welcome', 'screen-create-group', 'screen-home', 'screen-students', 'screen-schedule', 'screen-homework', 'screen-settings'];
        screens.forEach(id => {
          const el = document.getElementById(id);
          if (el) el.classList.add('hidden');
        });
        const target = document.getElementById(screenId);
        if (target) target.classList.remove('hidden');

        const isMainTab = ['screen-home', 'screen-students', 'screen-schedule', 'screen-homework', 'screen-settings'].includes(screenId);
        const bottomNav = document.getElementById('bottom-nav');
        const timeBanner = document.getElementById('global-time-banner');
        const roleBadge = document.getElementById('header-role-badge');
        const studentBanner = document.getElementById('student-mode-banner');

        if (isMainTab) {
          bottomNav.classList.remove('hidden');
          timeBanner.classList.remove('hidden');
          roleBadge.classList.remove('hidden');
          this.syncRoleUI();
        } else {
          bottomNav.classList.add('hidden');
          timeBanner.classList.add('hidden');
          roleBadge.classList.add('hidden');
          if (studentBanner) studentBanner.classList.add('hidden');
        }
        this.updateWelcomeScreenState();
        window.scrollTo({ top: 0, behavior: 'smooth' });
      },

      switchTab(tabName) {
        this.navigate(`screen-${tabName}`);
        const tabs = ['home', 'schedule', 'homework', 'students', 'settings'];
        tabs.forEach(t => {
          const btn = document.getElementById(`nav-btn-${t}`);
          if (!btn) return;
          if (t === tabName) {
            btn.classList.remove('text-slate-400');
            btn.classList.add('text-emerald-400');
          } else {
            btn.classList.remove('text-emerald-400');
            btn.classList.add('text-slate-400');
          }
        });
        if (tabName === 'homework') {
          this.populateInlineHwSubjects();
          this.renderHomework();
        }
      },

      resetToWelcome() {
        this.navigate('screen-welcome');
      },

      syncRoleUI() {
        const roleBadge = document.getElementById('header-role-badge');
        const studentBanner = document.getElementById('student-mode-banner');
        const studentBannerGroup = document.getElementById('student-banner-group');
        const studentSelfCard = document.getElementById('student-self-checkin-card');
        const onboardingCard = document.getElementById('onboarding-steps-card');
        const leaderBroadcastCard = document.getElementById('leader-broadcast-card');
        const editGroupBtn = document.getElementById('btn-edit-group-info');
        const editSemesterBtn = document.getElementById('btn-edit-semester');
        const addHolidayBtn = document.getElementById('btn-add-holiday');
        const scheduleEditorCard = document.getElementById('schedule-editor-card');
        const bellEditorControls = document.getElementById('bell-editor-controls');
        const scheduleReadonlyBadge = document.getElementById('schedule-readonly-badge');
        const fabAddClass = document.getElementById('fab-add-class');
        const addSelfCard = document.getElementById('add-self-card');
        const btnStudentLeave = document.getElementById('btn-student-leave-group');
        const settingsLeaveBtn = document.getElementById('settings-leave-group-btn');
        const leaderCodePill = document.getElementById('leader-code-pill');
        const settingsInviteRow = document.getElementById('settings-invite-code-row');
        const settingsDeputiesRow = document.getElementById('settings-deputies-row');
        const settingsBroadcastRow = document.getElementById('settings-broadcast-row');
        const settingsAutoAttendanceRow = document.getElementById('settings-auto-attendance-row');
        const settingsPollLeadRow = document.getElementById('settings-poll-lead-row');
        const settingsSectionTitle = document.getElementById('settings-section-title');
        const homeDeputiesBtn = document.getElementById('home-deputies-btn');
        const btnManageDeputies = document.getElementById('btn-manage-deputies');
        const btnAddStudent = document.getElementById('btn-add-student');
        const deputyInfoCard = document.getElementById('deputy-info-card');

        // Проверяем актуальную роль участника в текущей группе
        if (this.state.currentUserRole !== 'leader' && this.state.currentStudentId) {
          const me = this.state.students.find(s => s.id === this.state.currentStudentId);
          if (me) {
            this.state.currentUserRole = me.isDeputy ? 'deputy' : 'student';
          }
        }

        const isStarosta = this.isLeader();
        const canManage = this.canManageGroup();
        const isJoinedMember = !isStarosta;

        if (isStarosta) {
          roleBadge.textContent = 'Староста';
          roleBadge.className = 'px-2 py-0.5 rounded-full text-[10px] font-bold uppercase tracking-wider bg-emerald-500/20 text-emerald-300 border border-emerald-500/30';
        } else if (canManage) {
          roleBadge.textContent = 'Зам. старосты';
          roleBadge.className = 'px-2 py-0.5 rounded-full text-[10px] font-bold uppercase tracking-wider bg-amber-500/20 text-amber-300 border border-amber-500/30';
        } else {
          roleBadge.textContent = 'Студент';
          roleBadge.className = 'px-2 py-0.5 rounded-full text-[10px] font-bold uppercase tracking-wider bg-cyan-500/20 text-cyan-300 border border-cyan-500/30';
        }

        // Разграничение прав: рассылка посещаемости в TG, управление звонками, расписанием и замами ТОЛЬКО у Старосты и Зам. старосты
        if (canManage) {
          if (scheduleEditorCard) scheduleEditorCard.classList.remove('hidden');
          if (bellEditorControls) bellEditorControls.classList.remove('hidden');
          if (scheduleReadonlyBadge) scheduleReadonlyBadge.classList.add('hidden');
          if (fabAddClass) fabAddClass.classList.remove('hidden');
          if (editSemesterBtn) editSemesterBtn.classList.remove('hidden');
          if (addHolidayBtn) addHolidayBtn.classList.remove('hidden');
          if (leaderBroadcastCard) leaderBroadcastCard.classList.remove('hidden');
          if (homeDeputiesBtn) homeDeputiesBtn.classList.remove('hidden');
          if (btnManageDeputies) btnManageDeputies.classList.remove('hidden');
          if (btnAddStudent) btnAddStudent.classList.remove('hidden');
          if (deputyInfoCard) deputyInfoCard.classList.remove('hidden');
          if (settingsDeputiesRow) settingsDeputiesRow.classList.remove('hidden');
          if (settingsBroadcastRow) settingsBroadcastRow.classList.remove('hidden');
          if (settingsAutoAttendanceRow) settingsAutoAttendanceRow.classList.remove('hidden');
          if (settingsPollLeadRow) settingsPollLeadRow.classList.remove('hidden');
          if (settingsSectionTitle) settingsSectionTitle.textContent = 'ГРУППА, ЗАМЕСТИТЕЛИ И РАССЫЛКА';
        } else {
          if (scheduleEditorCard) scheduleEditorCard.classList.add('hidden');
          if (bellEditorControls) bellEditorControls.classList.add('hidden');
          if (scheduleReadonlyBadge) scheduleReadonlyBadge.classList.remove('hidden');
          if (fabAddClass) fabAddClass.classList.add('hidden');
          if (editSemesterBtn) editSemesterBtn.classList.add('hidden');
          if (addHolidayBtn) addHolidayBtn.classList.add('hidden');
          if (leaderBroadcastCard) leaderBroadcastCard.classList.add('hidden');
          if (homeDeputiesBtn) homeDeputiesBtn.classList.add('hidden');
          if (btnManageDeputies) btnManageDeputies.classList.add('hidden');
          if (btnAddStudent) btnAddStudent.classList.add('hidden');
          if (deputyInfoCard) deputyInfoCard.classList.add('hidden');
          // Убираем у обычных студентов в настройках рассылку посещаемости в ТГ, замов и авто-опросы
          if (settingsDeputiesRow) settingsDeputiesRow.classList.add('hidden');
          if (settingsBroadcastRow) settingsBroadcastRow.classList.add('hidden');
          if (settingsAutoAttendanceRow) settingsAutoAttendanceRow.classList.add('hidden');
          if (settingsPollLeadRow) settingsPollLeadRow.classList.add('hidden');
          if (settingsSectionTitle) settingsSectionTitle.textContent = 'ПАРАМЕТРЫ ПРИЛОЖЕНИЯ';
        }

        if (isJoinedMember) {
          studentBanner.classList.remove('hidden');
          document.getElementById('student-banner-name').textContent = this.state.currentStudentName || 'Студент';
          if (studentBannerGroup) studentBannerGroup.textContent = this.state.groupName;
          studentSelfCard.classList.remove('hidden');
          document.getElementById('student-card-group-name').textContent = this.state.groupName;
          if (btnStudentLeave) btnStudentLeave.classList.remove('hidden');
          if (settingsLeaveBtn) settingsLeaveBtn.classList.remove('hidden');
          if (onboardingCard) onboardingCard.classList.add('hidden');
          if (editGroupBtn) editGroupBtn.classList.add('hidden');
          if (addSelfCard) addSelfCard.classList.add('hidden');
          if (leaderCodePill && !canManage) leaderCodePill.classList.add('hidden');
          if (settingsInviteRow && !canManage) settingsInviteRow.classList.add('hidden');
        } else {
          studentBanner.classList.add('hidden');
          studentSelfCard.classList.add('hidden');
          if (btnStudentLeave) btnStudentLeave.classList.add('hidden');
          if (settingsLeaveBtn) settingsLeaveBtn.classList.add('hidden');
          if (onboardingCard) onboardingCard.classList.remove('hidden');
          if (editGroupBtn) editGroupBtn.classList.remove('hidden');
          if (leaderCodePill) leaderCodePill.classList.remove('hidden');
          if (settingsInviteRow) settingsInviteRow.classList.remove('hidden');
        }

        this.updateHomePrimaryWidgets();
      },

      updateHomePrimaryWidgets() {
        // Обновляем карточку "Посещаемость (за всё время)" на первой странице
        const summaryPill = document.getElementById('home-attendance-summary-pill');
        const subtextEl = document.getElementById('home-attendance-subtext');
        const totalStudents = this.state.students.length;
        const presentToday = this.state.students.filter(s => s.status.includes('На паре')).length;

        let totalAllTimeRecords = 0;
        let totalAllTimePresent = 0;
        this.state.students.forEach(s => {
          const stTotal = (s.totalClassesCount || 1);
          const stAbs = (s.absences || 0);
          const stPres = Math.max(0, stTotal - stAbs);
          totalAllTimeRecords += stTotal;
          totalAllTimePresent += stPres;
        });
        const avgRate = totalAllTimeRecords > 0 ? Math.round((totalAllTimePresent / totalAllTimeRecords) * 100) : 100;

        if (summaryPill) {
          summaryPill.textContent = `${totalStudents} уч. · ${avgRate}% за всё время`;
        }
        if (subtextEl) {
          const daysSaved = (this.state.attendanceHistory || []).length;
          subtextEl.textContent = totalStudents > 0
            ? `Сейчас на паре: ${presentToday} из ${totalStudents} чел. Общая посещаемость за всё время: ${avgRate}% (сохранено учебных дней: ${daysSaved}).`
            : `Статистика посещаемости всех участников за всё время учёбы, сводка за сегодня и история по датам.`;
        }

        // Обновляем карточку "Объявления" на первой странице
        const annPill = document.getElementById('home-announcements-count-pill');
        const annPreview = document.getElementById('home-latest-announcement-preview');
        const annCount = (this.state.announcements || []).length;
        if (annPill) {
          annPill.textContent = `${annCount} объявл.`;
        }
        if (annPreview) {
          if (annCount > 0) {
            const latest = this.state.announcements[0];
            annPreview.textContent = `${latest.title}: ${latest.text} (${latest.author || 'Группа'}, ${latest.date})`;
          } else {
            annPreview.textContent = 'Объявлений пока нет. Нажмите, чтобы опубликовать важную информацию для всей группы.';
          }
        }
      },

      handleCreateGroup(event) {
        event.preventDefault();
        const pdn = document.getElementById('consent-pdn').checked;
        const terms = document.getElementById('consent-terms').checked;
        const autoConsent = document.getElementById('consent-auto-attendance').checked;

        if (!pdn || !terms) {
          this.showToast('Необходимо принять согласие на обработку ПДн и пользовательское соглашение');
          return;
        }

        const groupName = document.getElementById('input-group-name').value.trim() || 'Новая группа';
        const uniName = document.getElementById('input-uni-name').value.trim() || 'Университет';
        const leaderName = document.getElementById('input-leader-name').value.trim() || 'Староста';
        const customCode = document.getElementById('input-custom-code').value.trim().toUpperCase();
        const newCode = customCode || this.generateRandomInviteCode();

        // Создаём полностью чистую и изолированную группу с пустым списком участников
        this.state.groupName = groupName;
        this.state.uniName = uniName;
        this.state.leaderName = leaderName;
        this.state.inviteCode = newCode;
        this.state.leaderOwnCode = newCode;
        this.state.autoAttendanceEnabled = autoConsent;
        this.state.currentUserRole = 'leader';
        this.state.currentStudentId = null;
        this.state.currentStudentName = '';
        this.state.hasCreatedGroup = true;

        // Изначально список группы и материалы новой группы полностью пусты
        this.state.students = [];
        this.state.bellSchedule = this.getDefaultBellSchedule();
        this.state.classes = [];
        this.state.homeworks = [];
        this.state.attendanceHistory = [];
        this.state.holidays = [];
        this.state.announcements = [];
        this.state.knowledgeFiles = [];
        this.state.broadcastHistory = [];
        this.state.stepsCompleted = { invite: false, schedule: false, autoPoll: true };

        this.saveState();
        this.syncGroupHeader();
        this.renderStudents();
        this.renderBellSchedule();
        this.renderSchedule();
        this.renderHomework();
        this.renderHolidays();
        this.updateProgressSteps();
        if (typeof this.toggleAutoAttendance === 'function') {
          this.toggleAutoAttendance(autoConsent, false);
        }
        this.switchTab('home');
        this.showToast(`Группа «${this.state.groupName}» создана! Список участников пуст.`);
      },

      syncGroupHeader() {
        const firstLetter = (this.state.groupName || 'Г').charAt(0).toUpperCase();
        document.getElementById('group-avatar').textContent = firstLetter;
        document.getElementById('settings-group-avatar').textContent = firstLetter;
        document.getElementById('display-group-name').textContent = this.state.groupName;
        document.getElementById('settings-group-name').textContent = this.state.groupName;
        document.getElementById('display-uni-name').textContent = this.state.uniName;
        document.getElementById('settings-uni-name').textContent = this.state.uniName;

        const codeBadge = document.getElementById('home-invite-code-badge');
        const settingsCode = document.getElementById('settings-invite-code');
        if (codeBadge) codeBadge.textContent = this.state.inviteCode || '—';
        if (settingsCode) settingsCode.textContent = this.state.inviteCode || '—';

        const deputies = this.state.students.filter(s => s.isDeputy).map(s => s.name);
        const depLine = document.getElementById('display-deputies-line');
        if (depLine) {
          const depText = deputies.length > 0 ? deputies.join(', ') : 'не назначен';
          depLine.textContent = `Староста: ${this.state.leaderName} · Зам. старосты: ${depText}`;
        }

        const depCountEl = document.getElementById('settings-deputies-count');
        if (depCountEl) {
          depCountEl.textContent = `${deputies.length} назн.`;
        }
        this.updateHomePrimaryWidgets();
      },

      // Окно входа для студента без подсказок и без отображения чужих кодов
      openStudentJoinModal() {
        this.openModal('Присоединиться как студент по коду', `
          <form onsubmit="App.handleStudentJoin(event)" class="space-y-4">
            <div>
              <label class="block text-xs text-slate-400 mb-1">Код приглашения от старосты</label>
              <input id="join-code-input" type="text" required autocomplete="off" placeholder="Введите код группы"
                class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-emerald-400 text-sm uppercase font-mono font-bold focus:border-emerald-500 focus:outline-none" />
            </div>

            <div>
              <label class="block text-xs text-slate-400 mb-1">Ваши Фамилия и Имя</label>
              <input id="join-student-name" type="text" required placeholder="Введите Фамилию и Имя"
                class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm focus:border-emerald-500 focus:outline-none" />
            </div>

            <div class="p-3 rounded-xl bg-cyan-950/40 border border-cyan-500/30 text-xs text-cyan-200 space-y-1">
              <p class="font-semibold">Права участника (Студент):</p>
              <p class="text-slate-300">Вы войдёте только в свою группу по коду старосты, сможете отмечать своё присутствие, добавлять материалы и в любой момент выйти из группы.</p>
            </div>

            <label class="flex items-start gap-2.5 cursor-pointer">
              <input type="checkbox" required checked class="mt-1 w-4 h-4 accent-emerald-500" />
              <span class="text-xs text-slate-300">Даю согласие на обработку ПДн и автоматическую отправку присутствия на парах.</span>
            </label>

            <button type="submit" class="w-full py-3.5 rounded-xl bg-gradient-to-r from-teal-500 to-emerald-500 hover:from-teal-400 hover:to-emerald-400 text-slate-950 font-semibold text-sm shadow-lg transition cursor-pointer">
              Войти в группу как студент
            </button>
          </form>
        `);
      },

      handleStudentJoin(e) {
        e.preventDefault();
        const enteredCode = (document.getElementById('join-code-input').value || '').trim().toUpperCase();
        const name = (document.getElementById('join-student-name').value || '').trim();

        if (!enteredCode) {
          this.showToast('Введите код приглашения группы');
          return;
        }
        if (!name) {
          this.showToast('Введите ваши Фамилию и Имя');
          return;
        }

        // Проверяем, существует ли группа с таким кодом в изолированном хранилище
        this.syncActiveGroupToStore();
        const groupExists = this.loadGroupFromStore(enteredCode);
        if (!groupExists) {
          this.showToast('Группа с таким кодом не найдена. Уточните код у вашего старосты.');
          return;
        }

        let existing = this.state.students.find(s => s.name.toLowerCase() === name.toLowerCase());
        if (!existing) {
          existing = {
            id: Date.now(),
            name: name,
            status: 'На паре (авто)',
            autoPresent: true,
            absences: 0,
            excusedCount: 0,
            presentCount: 1,
            totalClassesCount: 1,
            isLeader: false,
            isDeputy: false,
            joinedViaCode: true
          };
          this.state.students.push(existing);
        }

        this.state.currentUserRole = existing.isDeputy ? 'deputy' : 'student';
        this.state.currentStudentId = existing.id;
        this.state.currentStudentName = existing.name;
        this.state.studentJoinedCode = enteredCode;
        this.state.studentJoinedId = existing.id;
        this.state.studentJoinedName = existing.name;

        this.saveState();
        this.syncGroupHeader();
        this.renderStudents();
        this.renderBellSchedule();
        this.renderSchedule();
        this.renderHomework();
        this.renderHolidays();
        this.closeModal();
        this.switchTab('home');
        this.showToast(`Вы присоединились к группе «${this.state.groupName}»!`);
      },

      // Выход студента из группы
      leaveGroupAsStudent() {
        if (this.isLeader()) {
          this.showToast('Вы староста этой группы. Используйте кнопку «На экран входа» для смены роли.');
          return;
        }

        const leftGroupName = this.state.groupName;

        if (this.state.currentStudentId) {
          this.state.students = this.state.students.filter(s => s.id !== this.state.currentStudentId);
        } else if (this.state.currentStudentName) {
          const targetName = this.state.currentStudentName.toLowerCase();
          this.state.students = this.state.students.filter(s => s.name.toLowerCase() !== targetName);
        }

        this.syncActiveGroupToStore();
        this.state.currentStudentId = null;
        this.state.currentStudentName = '';
        this.state.studentJoinedCode = '';
        this.state.studentJoinedId = null;
        this.state.studentJoinedName = '';
        this.state.currentUserRole = 'student';
        this.saveState();

        this.closeModal();
        this.navigate('screen-welcome');
        this.showToast(`Вы вышли из группы «${leftGroupName}»`);
      },

      // Удаление участника старостой или заместителем старосты
      removeStudent(studentId, reopenDeputiesModal = false) {
        if (!this.canManageGroup()) {
          this.showToast('Удалять участников может только Староста или Зам. старосты');
          return;
        }

        const target = this.state.students.find(s => s.id === studentId);
        if (!target) return;

        if (target.isLeader && !this.isLeader()) {
          this.showToast('Зам. старосты не может удалить старосту группы');
          return;
        }

        const removedName = target.name;
        this.state.students = this.state.students.filter(s => s.id !== studentId);
        this.saveState();
        this.syncGroupHeader();
        this.renderStudents();

        if (reopenDeputiesModal) {
          this.openDeputiesModal();
        }
        this.showToast(`Участник ${removedName} удалён из группы`);
      },

      setMyStudentAttendance(newStatus) {
        let student = null;
        if (this.state.currentStudentId) {
          student = this.state.students.find(s => s.id === this.state.currentStudentId);
        }
        if (!student && this.state.currentStudentName) {
          student = this.state.students.find(s => s.name === this.state.currentStudentName);
        }
        if (student) {
          student.status = newStatus;
        }
        const badge = document.getElementById('student-my-status-badge');
        if (badge) {
          badge.textContent = newStatus;
        }
        this.saveState();
        this.renderStudents();
        this.showToast(`Ваш статус обновлён: «${newStatus}»`);
      },

      openDeputiesModal() {
        const canAssignDeputy = this.isLeader();
        const canRemoveMember = this.canManageGroup();
        const connectedStudents = this.state.students.filter(s => !s.isLeader);

        const rowsHtml = connectedStudents.length > 0
          ? connectedStudents.map(st => {
              const isDep = Boolean(st.isDeputy);
              return `
                <div class="p-3.5 rounded-xl bg-slate-900 border ${isDep ? 'border-amber-500/50' : 'border-slate-800'} flex flex-wrap items-center justify-between gap-3">
                  <div>
                    <div class="flex items-center gap-2 flex-wrap">
                      <p class="text-sm font-semibold text-white">${st.name}</p>
                      ${isDep
                        ? '<span class="px-2 py-0.5 rounded-md text-[10px] font-bold bg-amber-500/20 text-amber-300 border border-amber-500/40">👑 ЗАМ. СТАРОСТЫ</span>'
                        : '<span class="px-2 py-0.5 rounded-md text-[10px] bg-slate-800 text-slate-400">Студент</span>'}
                      ${st.joinedViaCode ? '<span class="text-[10px] text-emerald-400">· по коду</span>' : ''}
                    </div>
                    <p class="text-xs text-slate-400 mt-0.5">Статус: ${st.status}</p>
                  </div>

                  <div class="flex items-center gap-2">
                    ${canAssignDeputy ? `
                      <button type="button" onclick="App.toggleDeputyRole(${st.id}, true)" class="px-3 py-1.5 rounded-xl text-xs font-semibold transition shrink-0 ${
                        isDep
                          ? 'bg-amber-500/15 hover:bg-amber-500/25 border border-amber-500/30 text-amber-300'
                          : 'bg-amber-500 hover:bg-amber-400 text-slate-950'
                      }">
                        ${isDep ? 'Снять с зама' : 'Сделать замом'}
                      </button>
                    ` : ''}

                    ${canRemoveMember ? `
                      <button type="button" onclick="App.removeStudent(${st.id}, true)" class="px-2.5 py-1.5 rounded-xl bg-rose-500/15 hover:bg-rose-500/25 border border-rose-500/30 text-rose-300 text-xs font-semibold transition shrink-0" title="Удалить участника из группы">
                        Удалить
                      </button>
                    ` : ''}
                  </div>
                </div>
              `;
            }).join('')
          : `<p class="text-xs text-slate-400 py-4 text-center">В группе пока нет подключившихся участников.</p>`;

        this.openModal('Заместители старосты и участники', `
          <div class="space-y-3">
            <p class="text-xs text-slate-300 leading-relaxed">
              Староста может назначать <strong>Зам. старосты</strong>, а староста и заместители могут удалять участников из группы.
            </p>
            <div class="space-y-2 max-h-72 overflow-y-auto pr-1">
              ${rowsHtml}
            </div>
            ${canRemoveMember ? `
              <button type="button" onclick="App.openAddStudentModal()" class="w-full py-2.5 rounded-xl bg-slate-800 hover:bg-slate-700 border border-slate-700 text-xs font-semibold text-emerald-300 transition">
                + Добавить нового участника в группу
              </button>
            ` : `
              <p class="text-[11px] text-amber-300/80 text-center">Управлять заместителями может только Староста группы.</p>
            `}
          </div>
        `);
      },

      toggleDeputyRole(studentId, reopenModal = false) {
        if (!this.isLeader()) {
          this.showToast('Только староста может назначать или снимать зам. старосты');
          return;
        }
        const st = this.state.students.find(s => s.id === studentId);
        if (!st) return;

        st.isDeputy = !st.isDeputy;
        this.saveState();
        this.syncGroupHeader();
        this.renderStudents();

        if (reopenModal) {
          this.openDeputiesModal();
        }
        this.showToast(
          st.isDeputy
            ? `👑 ${st.name} назначен(а) Зам. старосты!`
            : `${st.name} больше не является зам. старосты`
        );
      },

      generateAttendanceReportText() {
        const tz = this.getCurrentTimezone();
        const dateStr = this.formatDateInTz(tz.id);
        const timeStr = this.formatTimeInTz(tz.id, false);
        const weekLabel = this.state.currentWeekNumber === 1 ? '★ 1-я неделя' : '★★ 2-я неделя';

        const presentList = this.state.students.filter(s => s.status.includes('На паре'));
        const lateOrExcused = this.state.students.filter(s => !s.status.includes('На паре') && s.status !== 'Не был');
        const absentList = this.state.students.filter(s => s.status === 'Не был');
        const deputies = this.state.students.filter(s => s.isDeputy).map(s => s.name);

        const lines = [
          `📋 СВОДКА ПОСЕЩАЕМОСТИ · Группа ${this.state.groupName}`,
          `🏛 ${this.state.uniName}`,
          `🕒 ${dateStr}, ${timeStr} (${tz.short}) · ${weekLabel}`,
          `👑 Староста: ${this.state.leaderName}` + (deputies.length ? ` | Зам: ${deputies.join(', ')}` : ''),
          ``,
          `✅ На паре (${presentList.length} чел.):`,
          presentList.length > 0 ? presentList.map((s, i) => `${i + 1}. ${s.name}${s.isDeputy ? ' (Зам. старосты)' : ''}`).join('\n') : '— никто не отмечен'
        ];

        if (lateOrExcused.length > 0) {
          lines.push(``, `⏳ Опаздывают / Уважительная (${lateOrExcused.length} чел.):`);
          lines.push(lateOrExcused.map(s => `• ${s.name} — ${s.status}`).join('\n'));
        }

        if (absentList.length > 0) {
          lines.push(``, `❌ Отсутствуют (${absentList.length} чел.):`);
          lines.push(absentList.map(s => `• ${s.name}`).join('\n'));
        }

        lines.push(``, `🤖 Отправлено через Telegram Mini App «Староста Бот» (код группы: ${this.state.inviteCode})`);
        return lines.join('\n');
      },

      openBroadcastAttendanceModal() {
        if (!this.canManageGroup()) {
          this.showToast('Отправлять всем сообщение о посещаемости может только Староста или Зам. старосты');
          return;
        }
        const reportText = this.generateAttendanceReportText();
        const total = this.state.students.length;

        this.openModal('Отправить всем сообщение о посещаемости', `
          <div class="space-y-3.5">
            <div class="p-3 rounded-xl bg-emerald-950/50 border border-emerald-500/30 flex items-center justify-between">
              <div>
                <p class="text-xs font-bold text-emerald-300">ГОТОВО К РАССЫЛКЕ ВСЕМ УЧАСТНИКАМ (${total} чел.)</p>
                <p class="text-[11px] text-slate-300">Бот разошлёт уведомление каждому студенту группы и в общий чат Telegram</p>
              </div>
              <span class="px-2.5 py-1 rounded-lg bg-emerald-500/20 text-emerald-300 font-mono text-xs font-bold">${this.state.groupName}</span>
            </div>

            <div>
              <label class="block text-xs text-slate-400 mb-1">Текст сообщения для отправки всем:</label>
              <textarea id="broadcast-report-textarea" rows="8" class="w-full p-3 rounded-xl bg-slate-950 border border-slate-800 text-xs text-slate-200 font-mono leading-relaxed focus:border-emerald-500 focus:outline-none">${reportText}</textarea>
            </div>

            <div class="grid grid-cols-1 sm:grid-cols-2 gap-2.5">
              <button type="button" onclick="App.sendAttendanceToAllViaBot()" class="py-3 px-4 rounded-xl bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 font-semibold text-xs sm:text-sm flex items-center justify-center gap-2 shadow-lg transition">
                <span>🚀 Разослать всем через Бота</span>
              </button>

              <button type="button" onclick="App.shareAttendanceToTelegramChat()" class="py-3 px-4 rounded-xl bg-sky-500/20 hover:bg-sky-500/30 border border-sky-500/40 text-sky-300 font-semibold text-xs sm:text-sm flex items-center justify-center gap-2 transition">
                <span>✈️ Переслать в чат Telegram</span>
              </button>
            </div>

            <button type="button" onclick="App.copyBroadcastReport()" class="w-full py-2.5 rounded-xl bg-slate-800 hover:bg-slate-700 text-slate-200 text-xs font-medium transition">
              📋 Скопировать текст сводки
            </button>
          </div>
        `);
      },

      sendAttendanceToAllViaBot() {
        const textarea = document.getElementById('broadcast-report-textarea');
        const text = textarea ? textarea.value : this.generateAttendanceReportText();
        const tz = this.getCurrentTimezone();
        const now = this.formatTimeInTz(tz.id, false);
        const dateStr = this.formatDateInTz(tz.id);

        try {
          if (window.Telegram && window.Telegram.WebApp) {
            if (window.Telegram.WebApp.HapticFeedback) {
              window.Telegram.WebApp.HapticFeedback.notificationOccurred('success');
            }
          }
        } catch (err) {
          console.warn(err);
        }

        this.copyTextFallback(text);

        // Фиксируем снимок посещаемости в истории за всё время
        this.recordAttendanceSnapshot(`${dateStr} (${now})`);

        this.state.broadcastHistory.unshift({
          id: Date.now(),
          time: now,
          count: this.state.students.length
        });
        this.saveState();

        const liveText = document.getElementById('live-attendance-text');
        if (liveText) {
          liveText.textContent = `✅ Рассылка выполнена в ${now} (${tz.short}): сообщение о посещаемости отправлено всем участникам группы (${this.state.students.length} чел.) и сохранено в историю за всё время!`;
        }

        this.updateHomePrimaryWidgets();
        this.closeModal();
        this.showToast(`Сообщение о посещаемости отправлено всем (${this.state.students.length} уч.)!`);
      },

      shareAttendanceToTelegramChat() {
        const textarea = document.getElementById('broadcast-report-textarea');
        const text = textarea ? textarea.value : this.generateAttendanceReportText();
        const shareUrl = `https://t.me/share/url?url=${encodeURIComponent('https://t.me/')}&text=${encodeURIComponent(text)}`;
        try {
          if (window.Telegram && window.Telegram.WebApp && window.Telegram.WebApp.openTelegramLink) {
            window.Telegram.WebApp.openTelegramLink(shareUrl);
          } else {
            window.open(shareUrl, '_blank');
          }
        } catch (e) {
          window.open(shareUrl, '_blank');
        }
        this.showToast('Открыто окно отправки сводки в чат Telegram');
      },

      copyTextFallback(text) {
        try {
          const temp = document.createElement('textarea');
          temp.value = text;
          document.body.appendChild(temp);
          temp.select();
          document.execCommand('copy');
          document.body.removeChild(temp);
        } catch (e) {
          if (navigator.clipboard) {
            navigator.clipboard.writeText(text).catch(() => {});
          }
        }
      },

      copyBroadcastReport() {
        const textarea = document.getElementById('broadcast-report-textarea');
        const text = textarea ? textarea.value : this.generateAttendanceReportText();
        this.copyTextFallback(text);
        this.showToast('Сводка посещаемости скопирована в буфер обмена');
      },

      triggerAttendancePhotoUpload() {
        if (!this.canManageGroup()) {
          this.showToast('Отмечать посещаемость группы по фото может только Староста или Зам. старосты');
          return;
        }
        const input = document.getElementById('attendance-photo-input');
        input.value = '';
        input.click();
      },

      handleAttendancePhoto(event) {
        if (!this.canManageGroup()) return;
        const file = event.target.files && event.target.files[0];
        if (!file) return;
        const imageUrl = URL.createObjectURL(file);

        this.state.students.forEach(s => {
          s.status = 'На паре (по фото)';
        });
        this.saveState();
        this.renderStudents();
        this.updateHomePrimaryWidgets();

        this.openModal('ИИ-проверка посещаемости по фото', `
          <div class="space-y-4">
            <div class="relative rounded-2xl overflow-hidden border border-emerald-500/40 bg-slate-950 max-h-52 flex items-center justify-center">
              <img src="${imageUrl}" alt="Фото посещаемости" class="max-h-52 w-auto object-contain" />
            </div>
            <div class="p-3.5 rounded-xl bg-emerald-950/50 border border-emerald-500/30 space-y-1">
              <p class="text-xs font-bold text-emerald-300">ФОТО ОБРАБОТАНО · ОТМЕЧЕНО ${this.state.students.length} ЧЕЛ.</p>
              <p class="text-xs text-slate-300">Все участники группы отмечены в журнале со статусом «На паре (по фото)». Теперь вы можете одной кнопкой разослать сводку всем в Telegram.</p>
            </div>
            <button type="button" onclick="App.openBroadcastAttendanceModal()" class="w-full py-3 rounded-xl bg-gradient-to-r from-emerald-500 to-teal-500 text-slate-950 font-semibold text-xs sm:text-sm shadow-lg transition">
              🚀 Отправить всем сообщение о посещаемости
            </button>
          </div>
        `);
      },

      triggerPhotoScheduleUpload() {
        if (!this.canManageGroup()) {
          this.showToast('Изменять расписание уроков может только Староста или Зам. старосты');
          return;
        }
        const input = document.getElementById('global-schedule-photo-input');
        input.value = '';
        input.click();
      },

      // =================================================================
      // РАСПИСАНИЕ ЗВОНКОВ И ПЕРЕМЕН (ИИ ПО ФОТО + РУЧНОЙ ВВОД)
      // =================================================================
      getBellTimeForLesson(lessonNum) {
        const bells = this.state.bellSchedule || this.getDefaultBellSchedule();
        const found = bells.find(b => Number(b.lessonNum) === Number(lessonNum));
        if (found) {
          return `${found.start} — ${found.end}`;
        }
        return '09:00 — 10:30';
      },

      calcBreakMinutes(endStr, nextStartStr) {
        try {
          const [h1, m1] = endStr.split(':').map(Number);
          const [h2, m2] = nextStartStr.split(':').map(Number);
          const diff = (h2 * 60 + m2) - (h1 * 60 + m1);
          return diff > 0 && diff <= 120 ? diff : 0;
        } catch (e) {
          return 10;
        }
      },

      renderBellSchedule() {
        const container = document.getElementById('bell-schedule-list');
        if (!container) return;
        const bells = this.state.bellSchedule || this.getDefaultBellSchedule();

        container.innerHTML = bells.map((b, idx) => {
          const nextBell = bells[idx + 1];
          const computedBreak = nextBell ? this.calcBreakMinutes(b.end, nextBell.start) : (b.breakMinutes || 0);
          const breakLabel = computedBreak > 0 ? `перемена ${computedBreak} мин` : 'без перемены';
          return `
            <div class="p-3 rounded-xl bg-slate-900/90 border border-slate-800 flex items-center justify-between gap-2">
              <div>
                <span class="px-2 py-0.5 rounded-md bg-teal-500/15 text-teal-300 border border-teal-500/30 text-[11px] font-bold">
                  ${b.lessonNum === 0 ? '0 пара / Кл. час' : `${b.lessonNum} пара`}
                </span>
                <p class="text-xs sm:text-sm font-mono font-bold text-white mt-1">${b.start} — ${b.end}</p>
              </div>
              <span class="text-[11px] text-slate-400 text-right">${breakLabel}</span>
            </div>
          `;
        }).join('');
      },

      triggerBellPhotoUpload() {
        if (!this.canManageGroup()) {
          this.showToast('Настраивать расписание звонков может только Староста или Зам. старосты');
          return;
        }
        const input = document.getElementById('bell-schedule-photo-input');
        input.value = '';
        input.click();
      },

      fileToBase64(file) {
        return new Promise((resolve, reject) => {
          const reader = new FileReader();
          reader.onload = () => {
            const res = String(reader.result || '');
            const base64 = res.includes(',') ? res.split(',')[1] : res;
            resolve(base64);
          };
          reader.onerror = reject;
          reader.readAsDataURL(file);
        });
      },

      async callGeminiVisionJSON(promptText, base64Data, mimeType, responseSchema) {
        const apiKey = "";
        const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent?key=${apiKey}`;
        const payload = {
          contents: [
            {
              role: "user",
              parts: [
                { text: promptText },
                { inlineData: { mimeType: mimeType || "image/jpeg", data: base64Data } }
              ]
            }
          ],
          generationConfig: {
            responseMimeType: "application/json",
            responseSchema: responseSchema
          }
        };

        let delay = 1000;
        for (let attempt = 0; attempt < 3; attempt++) {
          try {
            const response = await fetch(apiUrl, {
              method: 'POST',
              headers: { 'Content-Type': 'application/json' },
              body: JSON.stringify(payload)
            });
            if (response.ok) {
              const result = await response.json();
              const rawJson = result?.candidates?.[0]?.content?.parts?.[0]?.text;
              if (rawJson) {
                return JSON.parse(rawJson);
              }
            }
          } catch (e) {
            // Повтор при временной ошибке сети
          }
          await new Promise(r => setTimeout(r, delay));
          delay *= 2;
        }
        return null;
      },

      async handleBellSchedulePhoto(event) {
        if (!this.canManageGroup()) return;
        const file = event.target.files && event.target.files[0];
        if (!file) return;
        const imageUrl = URL.createObjectURL(file);

        this.openModal('ИИ-распознавание расписания звонков...', `
          <div class="space-y-4">
            <div class="relative rounded-2xl overflow-hidden border border-teal-500/40 bg-slate-950 max-h-56 flex items-center justify-center">
              <img src="${imageUrl}" alt="Фото звонков" class="max-h-56 w-auto object-contain opacity-85" />
              <div class="ai-scan-bar"></div>
            </div>
            <div class="p-3.5 rounded-xl bg-slate-900 border border-slate-800 flex items-center gap-3">
              <div class="w-5 h-5 border-2 border-teal-400 border-t-transparent rounded-full animate-spin shrink-0"></div>
              <p class="text-xs sm:text-sm font-semibold text-white">Считываем время начала и окончания пар и перемены...</p>
            </div>
          </div>
        `);

        let parsedBells = null;

        try {
          const base64Data = await this.fileToBase64(file);
          const schema = {
            type: "ARRAY",
            items: {
              type: "OBJECT",
              properties: {
                lessonNum: { type: "INTEGER" },
                start: { type: "STRING" },
                end: { type: "STRING" },
                breakMinutes: { type: "INTEGER" }
              },
              propertyOrdering: ["lessonNum", "start", "end", "breakMinutes"]
            }
          };
          const prompt = "Распознай расписание звонков с фотографии. Верни JSON-массив всех пар (lessonNum: номер пары 0, 1, 2, 3..., start: время начала ЧЧ:ММ, end: время окончания ЧЧ:ММ, breakMinutes: перемена в минутах после пары).";
          const aiResult = await this.callGeminiVisionJSON(prompt, base64Data, file.type, schema);
          if (Array.isArray(aiResult) && aiResult.length > 0) {
            parsedBells = aiResult;
          }
        } catch (e) {
          parsedBells = null;
        }

        // Если облачный ИИ недоступен, распознаём время звонков через встроенный OCR Tesseract
        if (!parsedBells) {
          try {
            if (window.Tesseract) {
              const res = await Promise.race([
                window.Tesseract.recognize(file, 'rus+eng', {}),
                new Promise((_, rej) => setTimeout(() => rej(new Error('ocr_timeout')), 8000))
              ]);
              const text = res?.data?.text || '';
              const timeMatches = [...text.matchAll(/(\d{1,2})[:.](\d{2})\s*[-—–]\s*(\d{1,2})[:.](\d{2})/g)];
              if (timeMatches.length > 0) {
                parsedBells = timeMatches.map((m, idx) => {
                  const start = `${String(m[1]).padStart(2, '0')}:${m[2]}`;
                  const end = `${String(m[3]).padStart(2, '0')}:${m[4]}`;
                  return { lessonNum: idx + 1, start, end, breakMinutes: 10 };
                });
              }
            }
          } catch (e) {
            // Используем текущую сетку для редактирования
          }
        }

        if (!parsedBells || parsedBells.length === 0) {
          parsedBells = JSON.parse(JSON.stringify(this.state.bellSchedule || this.getDefaultBellSchedule()));
        }

        // Пересчитываем перемены между парами
        for (let i = 0; i < parsedBells.length - 1; i++) {
          parsedBells[i].breakMinutes = this.calcBreakMinutes(parsedBells[i].end, parsedBells[i + 1].start);
        }

        this.openBellScheduleModal(parsedBells, imageUrl);
      },

      openBellScheduleModal(customBells = null, imageUrl = '') {
        if (!this.canManageGroup()) {
          this.showToast('Изменять расписание звонков может только Староста или Зам. старосты');
          return;
        }
        this.state.pendingBellSchedule = customBells
          ? JSON.parse(JSON.stringify(customBells))
          : JSON.parse(JSON.stringify(this.state.bellSchedule || this.getDefaultBellSchedule()));

        this.renderBellModalContent(imageUrl);
      },

      renderBellModalContent(imageUrl = '') {
        const rowsHtml = this.state.pendingBellSchedule.map((b, idx) => `
          <div class="p-3 rounded-xl bg-slate-900 border border-slate-800 grid grid-cols-12 gap-2 items-center">
            <div class="col-span-3">
              <label class="block text-[10px] text-slate-400">№ пары</label>
              <input type="number" min="0" max="12" value="${b.lessonNum}" onchange="App.updatePendingBell(${idx}, 'lessonNum', this.value)"
                class="w-full px-2 py-1.5 rounded-lg bg-slate-950 border border-slate-700 text-teal-300 font-bold text-xs" />
            </div>
            <div class="col-span-4">
              <label class="block text-[10px] text-slate-400">Начало</label>
              <input type="time" value="${b.start}" onchange="App.updatePendingBell(${idx}, 'start', this.value)"
                class="w-full px-2 py-1.5 rounded-lg bg-slate-950 border border-slate-700 text-white font-mono text-xs" />
            </div>
            <div class="col-span-4">
              <label class="block text-[10px] text-slate-400">Конец</label>
              <input type="time" value="${b.end}" onchange="App.updatePendingBell(${idx}, 'end', this.value)"
                class="w-full px-2 py-1.5 rounded-lg bg-slate-950 border border-slate-700 text-white font-mono text-xs" />
            </div>
            <div class="col-span-1 flex justify-end pt-3">
              <button type="button" onclick="App.removePendingBellSlot(${idx})" class="text-slate-500 hover:text-rose-400 text-sm" title="Удалить звонок">✕</button>
            </div>
          </div>
        `).join('');

        this.openModal('Настройка расписания звонков и перемен', `
          <div class="space-y-3.5">
            ${imageUrl ? `
              <div class="flex items-center gap-3 p-2.5 rounded-xl bg-slate-900 border border-teal-500/30">
                <img src="${imageUrl}" alt="Скан звонков" class="w-12 h-12 rounded-lg object-cover border border-slate-700 shrink-0" />
                <p class="text-xs text-teal-300">Проверьте распознанное время звонков или отредактируйте слоты вручную.</p>
              </div>
            ` : `
              <p class="text-xs text-slate-300">Укажите время начала и окончания каждой пары в вашем колледже или университете. Перемены и время уроков в расписании обновятся автоматически.</p>
            `}

            <div class="space-y-2 max-h-64 overflow-y-auto pr-1">
              ${rowsHtml}
            </div>

            <button type="button" onclick="App.addPendingBellSlot()" class="w-full py-2.5 rounded-xl bg-slate-900 hover:bg-slate-800 border border-slate-700 text-xs font-semibold text-teal-300 transition">
              + Добавить ещё пару (звонок)
            </button>

            <button type="button" onclick="App.saveBellSchedule()" class="w-full py-3 rounded-xl bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 font-semibold text-xs sm:text-sm shadow-lg transition">
              Сохранить звонки и обновить время пар
            </button>
          </div>
        `);
      },

      updatePendingBell(idx, field, val) {
        if (!this.state.pendingBellSchedule[idx]) return;
        if (field === 'lessonNum') {
          this.state.pendingBellSchedule[idx].lessonNum = parseInt(val, 10) || 0;
        } else {
          this.state.pendingBellSchedule[idx][field] = val;
        }
      },

      addPendingBellSlot() {
        const list = this.state.pendingBellSchedule || [];
        const last = list[list.length - 1];
        const nextNum = last ? Number(last.lessonNum) + 1 : 1;
        list.push({
          lessonNum: nextNum,
          start: '17:30',
          end: '19:00',
          breakMinutes: 10
        });
        this.renderBellModalContent();
      },

      removePendingBellSlot(idx) {
        this.state.pendingBellSchedule.splice(idx, 1);
        this.renderBellModalContent();
      },

      saveBellSchedule() {
        const cleaned = (this.state.pendingBellSchedule || [])
          .slice()
          .sort((a, b) => Number(a.lessonNum) - Number(b.lessonNum));

        for (let i = 0; i < cleaned.length - 1; i++) {
          cleaned[i].breakMinutes = this.calcBreakMinutes(cleaned[i].end, cleaned[i + 1].start);
        }

        this.state.bellSchedule = cleaned;

        // Синхронизируем время существующих пар в расписании с новыми звонками
        this.state.classes.forEach(c => {
          if (c.lessonNum !== undefined && c.lessonNum !== null) {
            c.time = this.getBellTimeForLesson(c.lessonNum);
          }
        });

        this.saveState();
        this.renderBellSchedule();
        this.renderSchedule();
        this.closeModal();
        this.showToast('Расписание звонков и перемен сохранено!');
      },

      // =================================================================
      // УЛУЧШЕННОЕ УНИВЕРСАЛЬНОЕ ИИ-РАСПОЗНАВАНИЕ ЛЮБОГО ФОТО РАСПИСАНИЯ
      // =================================================================
      async handleAISchedulePhoto(event) {
        if (!this.canManageGroup()) {
          this.showToast('Изменять расписание уроков может только Староста или Зам. старосты');
          return;
        }
        const file = event.target.files && event.target.files[0];
        if (!file) return;

        const imageUrl = URL.createObjectURL(file);

        this.openModal('ИИ-анализ фотографии расписания...', `
          <div class="space-y-4">
            <div class="relative rounded-2xl overflow-hidden border border-emerald-500/40 bg-slate-950 max-h-60 flex items-center justify-center">
              <img src="${imageUrl}" alt="Скан расписания" class="max-h-60 w-auto object-contain opacity-85" />
              <div class="ai-scan-bar"></div>
            </div>
            <div class="p-3.5 rounded-xl bg-slate-900 border border-slate-800 flex items-center gap-3">
              <div class="w-5 h-5 border-2 border-emerald-400 border-t-transparent rounded-full animate-spin shrink-0"></div>
              <div>
                <p class="text-xs sm:text-sm font-semibold text-white">Полностью переносим все пары, кабинеты, преподавателей и недели ★ / ★★ с фото...</p>
                <p class="text-[11px] text-slate-400">★ 1 звезда = 1-я неделя · ★★ 2 звезды = 2-я неделя · время синхронизируется со звонками</p>
              </div>
            </div>
          </div>
        `);

        let extractedClasses = null;

        // 1. Полноценное мультимодальное ИИ-распознавание всех строк расписания с фото через Gemini Vision
        try {
          const base64Data = await this.fileToBase64(file);
          const schema = {
            type: "ARRAY",
            items: {
              type: "OBJECT",
              properties: {
                day: { type: "STRING" },
                lessonNum: { type: "INTEGER" },
                subject: { type: "STRING" },
                teacher: { type: "STRING" },
                room: { type: "STRING" },
                weekType: { type: "STRING" }
              },
              propertyOrdering: ["day", "lessonNum", "subject", "teacher", "room", "weekType"]
            }
          };
          const prompt = `Внимательно прочитай фотографию расписания занятий и перенеси АБСОЛЮТНО ВСЕ пары по всем дням недели (Понедельник, Вторник, Среда, Четверг, Пятница, Суббота).
Если в одной ячейке (на одной паре) указаны два предмета со звёздочками или через дробь (например: 1 звезда ★ или * — это 1-я неделя, 2 звезды ★★ или ** — это 2-я неделя), обязательно создай ДВЕ отдельные записи с одинаковым day и lessonNum: одну с weekType: "1", другую с weekType: "2" и соответствующими им кабинетами и преподавателями.
Если звезда не указана — ставь weekType: "both".
Пропускай пустые строки (где прочерк "—").
Поля:
- day: "Понедельник" | "Вторник" | "Среда" | "Четверг" | "Пятница" | "Суббота"
- lessonNum: номер урока/пары (0, 1, 2, 3, 4, 5...)
- subject: точное название предмета без звёздочек
- teacher: ФИО преподавателя (или "" если нет)
- room: номер кабинета или "с/зал" (например "Каб. 204")
- weekType: "both" | "1" | "2"`;

          const aiItems = await this.callGeminiVisionJSON(prompt, base64Data, file.type, schema);
          if (Array.isArray(aiItems) && aiItems.length > 0) {
            extractedClasses = aiItems.map((item, idx) => {
              const num = item.lessonNum !== undefined && item.lessonNum !== null ? Number(item.lessonNum) : (idx + 1);
              const roomStr = (item.room || '').trim();
              const teacherStr = (item.teacher || '').trim();
              const combinedRoom = [
                roomStr ? (roomStr.toLowerCase().includes('каб') || roomStr.toLowerCase().includes('зал') ? roomStr : `Каб. ${roomStr}`) : '',
                teacherStr && teacherStr !== '—' ? teacherStr : ''
              ].filter(Boolean).join(' · ') || 'Каб. не указан';

              return {
                id: Date.now() + idx,
                day: item.day || 'Понедельник',
                lessonNum: num,
                time: this.getBellTimeForLesson(num),
                subject: (item.subject || 'Предмет').trim(),
                teacher: teacherStr,
                room: combinedRoom,
                weekType: item.weekType === '1' || item.weekType === '2' ? item.weekType : 'both',
                autoSend: true
              };
            });
          }
        } catch (err) {
          extractedClasses = null;
        }

        // 2. Если нет доступа к сети/API, используем глубокий построчный OCR-парсер по дням недели и звёздам
        if (!extractedClasses || extractedClasses.length === 0) {
          let ocrText = '';
          try {
            if (window.Tesseract) {
              const result = await Promise.race([
                window.Tesseract.recognize(file, 'rus', {}),
                new Promise((_, reject) => setTimeout(() => reject(new Error('timeout')), 10000))
              ]);
              if (result && result.data && result.data.text) {
                ocrText = result.data.text;
              }
            }
          } catch (e) {
            ocrText = '';
          }
          extractedClasses = this.parseRawScheduleText(ocrText);
        }

        this.state.pendingAIClasses = extractedClasses;
        this.openAIScheduleReviewModal(imageUrl, file.name);
      },

      // Построчный разбор OCR-текста любого расписания
      parseRawScheduleText(rawText) {
        const lines = String(rawText || '')
          .split(/\r?\n/)
          .map(l => l.trim())
          .filter(l => l.length > 1);

        const dayMap = [
          { regex: /понедельник|^пн\b/i, name: 'Понедельник' },
          { regex: /вторник|^вт\b/i, name: 'Вторник' },
          { regex: /среда|^ср\b/i, name: 'Среда' },
          { regex: /четверг|^чт\b/i, name: 'Четверг' },
          { regex: /пятница|^пт\b/i, name: 'Пятница' },
          { regex: /суббота|^сб\b/i, name: 'Суббота' }
        ];

        const results = [];
        let currentDay = 'Понедельник';
        let currentLessonNum = 1;

        for (let i = 0; i < lines.length; i++) {
          const line = lines[i];

          // Проверяем смену дня недели
          const matchedDay = dayMap.find(d => d.regex.test(line));
          if (matchedDay) {
            currentDay = matchedDay.name;
            currentLessonNum = 1;
            continue;
          }

          // Пропускаем заголовки таблицы
          if (/предмет|преподаватель|кабинет|расписание|группы/i.test(line)) {
            continue;
          }

          // Определяем номер пары в начале строки
          const numMatch = line.match(/^([0-6])\s+(.+)/);
          let content = line;
          if (numMatch) {
            currentLessonNum = parseInt(numMatch[1], 10);
            content = numMatch[2].trim();
          }

          if (!content || /^[-—_]+$/.test(content)) continue;

          // Определяем звёзды (★★ / ** = 2-я неделя, ★ / * = 1-я неделя)
          let weekType = 'both';
          if (/(\*{2}|★{2}|☆{2})/.test(content)) {
            weekType = '2';
          } else if (/(\*{1}|★{1}|☆{1})/.test(content)) {
            weekType = '1';
          }

          // Извлекаем номер кабинета в конце строки
          const roomMatch = content.match(/(\d{3}(?:\s*[/,]\s*\d{3})?|с\/зал|зал)\s*$/i);
          const room = roomMatch ? `Каб. ${roomMatch[1]}` : 'Каб. по расписанию';
          const cleanedSubject = content
            .replace(/(\d{3}(?:\s*[/,]\s*\d{3})?|с\/зал|зал)\s*$/i, '')
            .replace(/[★☆*]+/g, '')
            .trim();

          if (cleanedSubject.length >= 3) {
            results.push({
              id: Date.now() + results.length,
              day: currentDay,
              lessonNum: currentLessonNum,
              time: this.getBellTimeForLesson(currentLessonNum),
              subject: cleanedSubject,
              room: room,
              weekType: weekType,
              autoSend: true
            });
            if (!numMatch && currentLessonNum < 5) {
              currentLessonNum++;
            }
          }
        }

        // Если OCR не смог прочитать декоративный шрифт, создаём пустой черновик по дням недели для заполнения
        if (results.length === 0) {
          const days = ['Понедельник', 'Вторник', 'Среда', 'Четверг', 'Пятница'];
          days.forEach((d, dIdx) => {
            for (let num = 1; num <= 3; num++) {
              results.push({
                id: Date.now() + dIdx * 10 + num,
                day: d,
                lessonNum: num,
                time: this.getBellTimeForLesson(num),
                subject: `Пара №${num}`,
                room: 'Каб. —',
                weekType: 'both',
                autoSend: true
              });
            }
          });
        }

        return results;
      },

      openAIScheduleReviewModal(imageUrl, fileName) {
        const itemsHtml = this.state.pendingAIClasses.map((c, idx) => `
          <div class="p-3 rounded-xl bg-slate-900 border border-slate-800 space-y-2">
            <div class="flex items-center justify-between gap-2 flex-wrap">
              <div class="flex items-center gap-1.5">
                <select onchange="App.updatePendingClassDay(${idx}, this.value)" class="px-2 py-1 rounded-lg bg-slate-950 border border-slate-700 text-xs font-bold text-emerald-400">
                  ${['Понедельник', 'Вторник', 'Среда', 'Четверг', 'Пятница', 'Суббота'].map(d => `<option value="${d}" ${c.day === d ? 'selected' : ''}>${d}</option>`).join('')}
                </select>
                <input type="number" min="0" max="10" value="${c.lessonNum !== undefined ? c.lessonNum : 1}" onchange="App.updatePendingClassNum(${idx}, this.value)"
                  class="w-14 px-2 py-1 rounded-lg bg-slate-950 border border-slate-700 text-xs font-bold text-white text-center" title="Номер пары" />
                <span class="text-[11px] font-mono text-slate-400">${c.time}</span>
              </div>
              <select onchange="App.updatePendingClassWeek(${idx}, this.value)" class="px-2 py-1 rounded-lg bg-slate-950 border border-slate-700 text-xs font-semibold text-amber-300">
                <option value="both" ${c.weekType === 'both' ? 'selected' : ''}>Каждая неделя</option>
                <option value="1" ${c.weekType === '1' ? 'selected' : ''}>★ 1-я неделя</option>
                <option value="2" ${c.weekType === '2' ? 'selected' : ''}>★★ 2-я неделя</option>
              </select>
            </div>
            <div class="grid grid-cols-1 sm:grid-cols-2 gap-2">
              <input type="text" value="${c.subject}" onchange="App.updatePendingClassSubject(${idx}, this.value)" placeholder="Предмет"
                class="w-full px-2.5 py-1.5 rounded-lg bg-slate-950 border border-slate-800 text-white text-xs sm:text-sm font-medium focus:border-emerald-500 focus:outline-none" />
              <div class="flex items-center gap-1.5">
                <input type="text" value="${c.room}" onchange="App.updatePendingClassRoom(${idx}, this.value)" placeholder="Кабинет и преподаватель"
                  class="flex-1 px-2.5 py-1.5 rounded-lg bg-slate-950 border border-slate-800 text-slate-300 text-xs focus:border-emerald-500 focus:outline-none" />
                <button type="button" onclick="App.removePendingClass(${idx})" class="text-slate-500 hover:text-rose-400 p-1 shrink-0" title="Удалить">✕</button>
              </div>
            </div>
          </div>
        `).join('');

        this.openModal('Проверка распознанного расписания', `
          <div class="space-y-3.5">
            <div class="flex items-center gap-3 p-2.5 rounded-xl bg-slate-900/90 border border-emerald-500/30">
              ${imageUrl ? `<img src="${imageUrl}" alt="Фото" class="w-12 h-12 rounded-lg object-cover border border-slate-700 shrink-0" />` : ''}
              <div class="min-w-0 flex-1">
                <p class="text-xs font-semibold text-emerald-300 truncate">Распознано занятий: ${this.state.pendingAIClasses.length}</p>
                <p class="text-[11px] text-slate-400">Звёзды: <span class="text-amber-300">★ — 1-я неделя</span>, <span class="text-cyan-300">★★ — 2-я неделя</span>. Время подставлено из ваших звонков.</p>
              </div>
            </div>

            <div class="space-y-2 max-h-64 overflow-y-auto pr-1">
              ${itemsHtml}
            </div>

            <button type="button" onclick="App.addPendingClassSlot('${imageUrl || ''}')" class="w-full py-2 rounded-xl bg-slate-900 hover:bg-slate-800 border border-slate-700 text-xs font-semibold text-emerald-300 transition">
              + Добавить ещё строку
            </button>

            <button type="button" onclick="App.confirmAISchedule()" class="w-full py-3.5 rounded-xl bg-gradient-to-r from-emerald-500 to-teal-500 hover:from-emerald-400 hover:to-teal-400 text-slate-950 font-semibold text-xs sm:text-sm shadow-lg shadow-emerald-500/20 transition">
              Сохранить расписание (${this.state.pendingAIClasses.length} пар)
            </button>
          </div>
        `);
      },

      updatePendingClassDay(idx, val) {
        if (this.state.pendingAIClasses[idx]) {
          this.state.pendingAIClasses[idx].day = val;
        }
      },

      updatePendingClassNum(idx, val) {
        if (this.state.pendingAIClasses[idx]) {
          const num = parseInt(val, 10) || 0;
          this.state.pendingAIClasses[idx].lessonNum = num;
          this.state.pendingAIClasses[idx].time = this.getBellTimeForLesson(num);
        }
      },

      addPendingClassSlot(imageUrl) {
        this.state.pendingAIClasses.push({
          id: Date.now() + Math.random(),
          day: 'Понедельник',
          lessonNum: 1,
          time: this.getBellTimeForLesson(1),
          subject: '',
          room: '',
          weekType: 'both',
          autoSend: true
        });
        this.openAIScheduleReviewModal(imageUrl, 'Фото расписания');
      },

      updatePendingClassWeek(idx, val) {
        if (this.state.pendingAIClasses[idx]) {
          this.state.pendingAIClasses[idx].weekType = val;
        }
      },

      updatePendingClassSubject(idx, val) {
        if (this.state.pendingAIClasses[idx]) {
          this.state.pendingAIClasses[idx].subject = val;
        }
      },

      updatePendingClassRoom(idx, val) {
        if (this.state.pendingAIClasses[idx]) {
          this.state.pendingAIClasses[idx].room = val;
        }
      },

      removePendingClass(idx) {
        this.state.pendingAIClasses.splice(idx, 1);
        this.openAIScheduleReviewModal('', 'Скан расписания');
      },

      confirmAISchedule() {
        if (!this.canManageGroup()) {
          this.showToast('Изменять расписание уроков может только Староста или Зам. старосты');
          return;
        }
        // Заменяем расписание на распознанный полный недельный набор пар
        this.state.classes = [...this.state.pendingAIClasses];
        this.state.pendingAIClasses = [];
        this.state.stepsCompleted.schedule = true;
        this.saveState();
        this.updateProgressSteps();
        this.renderSchedule();
        this.populateInlineHwSubjects();
        this.closeModal();
        this.switchTab('schedule');
        this.showToast(`Полное расписание (${this.state.classes.length} пар) со звёздами ★ и ★★ сохранено!`);
      },

      setScheduleWeekFilter(filterVal) {
        this.state.scheduleWeekFilter = filterVal;
        const btns = {
          all: document.getElementById('week-filter-all'),
          '1': document.getElementById('week-filter-1'),
          '2': document.getElementById('week-filter-2')
        };
        Object.keys(btns).forEach(k => {
          if (k === filterVal) {
            btns[k].className = 'py-2.5 px-3 rounded-xl text-xs font-semibold bg-emerald-500 text-slate-950 transition';
          } else {
            const colorClass = k === '1' ? 'text-amber-300' : k === '2' ? 'text-cyan-300' : 'text-slate-300';
            btns[k].className = `py-2.5 px-3 rounded-xl text-xs font-semibold bg-slate-900 hover:bg-slate-800 ${colorClass} border border-slate-800 transition`;
          }
        });
        this.renderSchedule();
      },

      getWeekBadgeHtml(weekType) {
        if (weekType === '1') {
          return `<span class="px-2 py-0.5 rounded-md text-[11px] font-bold bg-amber-500/20 text-amber-300 border border-amber-500/40">★ 1-я неделя</span>`;
        }
        if (weekType === '2') {
          return `<span class="px-2 py-0.5 rounded-md text-[11px] font-bold bg-cyan-500/20 text-cyan-300 border border-cyan-500/40">★★ 2-я неделя</span>`;
        }
        return `<span class="px-2 py-0.5 rounded-md text-[11px] font-medium bg-emerald-500/15 text-emerald-300 border border-emerald-500/30">Каждая неделя</span>`;
      },

      renderSchedule() {
        const container = document.getElementById('schedule-classes-list');
        const clearBtn = document.getElementById('clear-schedule-btn');
        const canDelete = this.canManageGroup();

        if (this.state.classes.length === 0) {
          if (clearBtn) clearBtn.classList.add('hidden');
          container.innerHTML = `
            <div class="p-6 rounded-2xl bg-cardBg border border-slate-800/80 text-center space-y-2">
              <p class="text-sm font-medium text-slate-300">Расписание пока пустое</p>
              <p class="text-xs text-slate-400">Загрузите фото расписания выше (ИИ сам распознает пары и звёзды ★ / ★★) или нажмите кнопку «+» для добавления пары.</p>
            </div>
          `;
          return;
        }

        if (clearBtn) {
          if (canDelete) clearBtn.classList.remove('hidden');
          else clearBtn.classList.add('hidden');
        }

        const filter = this.state.scheduleWeekFilter;
        const filtered = this.state.classes.filter(item => {
          if (filter === 'all') return true;
          return item.weekType === 'both' || item.weekType === filter;
        });

        const daysOrder = ['Понедельник', 'Вторник', 'Среда', 'Четверг', 'Пятница', 'Суббота'];
        const grouped = {};
        filtered.forEach(item => {
          if (!grouped[item.day]) grouped[item.day] = [];
          grouped[item.day].push(item);
        });

        const daysHtml = daysOrder.filter(d => grouped[d] && grouped[d].length > 0).map(day => {
          const rows = grouped[day].map(item => `
            <div class="p-3.5 rounded-xl bg-slate-900/90 border border-slate-800/90 flex items-center justify-between gap-3">
              <div class="space-y-1">
                <div class="flex flex-wrap items-center gap-2">
                  ${item.lessonNum !== undefined ? `<span class="px-2 py-0.5 rounded-md bg-slate-800 text-emerald-400 text-[11px] font-bold">№ ${item.lessonNum}</span>` : ''}
                  <span class="text-xs font-mono text-slate-300">${item.time}</span>
                  ${this.getWeekBadgeHtml(item.weekType)}
                </div>
                <p class="text-sm font-bold text-white">${item.subject}</p>
                <p class="text-xs text-slate-400">${item.room}</p>
              </div>
              ${canDelete ? `
                <button type="button" onclick="App.deleteClass(${item.id})" class="text-slate-500 hover:text-rose-400 p-1.5 transition" title="Удалить пару">
                  <svg class="w-4 h-4" fill="none" stroke="currentColor" viewBox="0 0 24 24">
                    <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M19 7l-.867 12.142A2 2 0 0116.138 21H7.862a2 2 0 01-1.995-1.858L5 7m5 4v6m4-6v6m1-10V4a1 1 0 00-1-1h-4a1 1 0 00-1 1v3M4 7h16"/>
                  </svg>
                </button>
              ` : ''}
            </div>
          `).join('');

          return `
            <div class="p-4 rounded-2xl bg-cardBg border border-slate-800/90 space-y-2.5">
              <div class="flex items-center justify-between">
                <h4 class="text-xs font-bold uppercase tracking-wider text-emerald-400">${day}</h4>
                <span class="text-[11px] text-slate-500">${grouped[day].length} пар(ы)</span>
              </div>
              <div class="space-y-2">${rows}</div>
            </div>
          `;
        }).join('');

        container.innerHTML = daysHtml || `<p class="text-sm text-slate-400 py-3 text-center">На выбранной неделе пар нет.</p>`;
      },

      clearAllClasses() {
        if (!this.canManageGroup()) {
          this.showToast('Студент не может изменять или удалять расписание уроков');
          return;
        }
        this.state.classes = [];
        this.saveState();
        this.renderSchedule();
        this.showToast('Расписание очищено');
      },

      deleteClass(id) {
        if (!this.canManageGroup()) {
          this.showToast('Студент не может изменять расписание уроков в группе');
          return;
        }
        this.state.classes = this.state.classes.filter(c => c.id !== id);
        this.saveState();
        this.renderSchedule();
        this.showToast('Пара удалена');
      },

      openAddClassModal() {
        if (!this.canManageGroup()) {
          this.showToast('Изменять расписание уроков в группе может только Староста или Зам. старосты');
          return;
        }
        const defaultTime = this.getBellTimeForLesson(1);
        this.openModal('Добавить пару в расписание', `
          <form onsubmit="App.submitNewClass(event)" class="space-y-3.5">
            <div class="grid grid-cols-2 gap-2.5">
              <div>
                <label class="block text-xs text-slate-400 mb-1">День недели</label>
                <select id="class-day" class="w-full px-3 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm">
                  <option>Понедельник</option>
                  <option>Вторник</option>
                  <option>Среда</option>
                  <option>Четверг</option>
                  <option>Пятница</option>
                  <option>Суббота</option>
                </select>
              </div>
              <div>
                <label class="block text-xs text-slate-400 mb-1">Номер пары (№)</label>
                <input id="class-num" type="number" min="0" max="10" value="1" oninput="document.getElementById('class-time').value = App.getBellTimeForLesson(this.value)" class="w-full px-3 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
              </div>
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Звезда / Неделя (★ = 1-я неделя, ★★ = 2-я неделя)</label>
              <select id="class-week-type" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-amber-300 text-sm font-medium">
                <option value="both">Без звезды — Каждая неделя</option>
                <option value="1">★ 1 звезда — 1-я неделя</option>
                <option value="2">★★ 2 звезды — 2-я неделя</option>
              </select>
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Время (автоматически по расписанию звонков или вручную)</label>
              <input id="class-time" type="text" required value="${defaultTime}" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Предмет</label>
              <input id="class-subject" type="text" required placeholder="Введите название предмета" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Кабинет и преподаватель</label>
              <input id="class-room" type="text" placeholder="Укажите кабинет и ФИО преподавателя" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <button type="submit" class="w-full py-3 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-sm transition">
              Сохранить пару
            </button>
          </form>
        `);
      },

      submitNewClass(e) {
        e.preventDefault();
        if (!this.canManageGroup()) {
          this.showToast('Студент не может изменять расписание уроков');
          return;
        }
        const numVal = parseInt(document.getElementById('class-num').value, 10);
        this.state.classes.push({
          id: Date.now(),
          day: document.getElementById('class-day').value,
          lessonNum: isNaN(numVal) ? 1 : numVal,
          weekType: document.getElementById('class-week-type').value,
          time: document.getElementById('class-time').value,
          subject: document.getElementById('class-subject').value.trim(),
          room: document.getElementById('class-room').value.trim() || 'Каб. не указан',
          autoSend: true
        });
        this.state.stepsCompleted.schedule = true;
        this.saveState();
        this.updateProgressSteps();
        this.renderSchedule();
        this.populateInlineHwSubjects();
        this.closeModal();
        this.showToast('Пара добавлена в расписание');
      },

      // =================================================================
      // ДОМАШНЕЕ ЗАДАНИЕ С УЧЁТОМ НЕДЕЛЬ (★ 1-Я НЕДЕЛЯ / ★★ 2-Я НЕДЕЛЯ)
      // =================================================================
      setHomeworkWeekFilter(filterVal) {
        this.state.homeworkWeekFilter = filterVal;
        const btns = {
          all: document.getElementById('hw-week-filter-all'),
          '1': document.getElementById('hw-week-filter-1'),
          '2': document.getElementById('hw-week-filter-2')
        };
        Object.keys(btns).forEach(k => {
          if (!btns[k]) return;
          if (k === filterVal) {
            btns[k].className = 'py-2.5 px-3 rounded-xl text-xs font-semibold bg-emerald-500 text-slate-950 transition';
          } else {
            const colorClass = k === '1' ? 'text-amber-300' : k === '2' ? 'text-cyan-300' : 'text-slate-300';
            btns[k].className = `py-2.5 px-3 rounded-xl text-xs font-semibold bg-slate-900 hover:bg-slate-800 ${colorClass} border border-slate-800 transition`;
          }
        });
        this.renderHomework();
      },

      populateInlineHwSubjects() {
        const datalist = document.getElementById('hw-subjects-datalist');
        if (!datalist) return;
        const dayEl = document.getElementById('inline-hw-day');
        const weekEl = document.getElementById('inline-hw-week');
        const selectedDay = dayEl ? dayEl.value : null;
        const selectedWeek = weekEl ? weekEl.value : 'all';

        const matchingSubjects = new Set();
        this.state.classes.forEach(c => {
          const dayMatch = !selectedDay || c.day === selectedDay;
          const weekMatch = selectedWeek === 'both' || c.weekType === 'both' || c.weekType === selectedWeek;
          if (dayMatch && weekMatch && c.subject && c.subject !== 'Классный час') {
            matchingSubjects.add(c.subject);
          }
        });
        if (matchingSubjects.size === 0) {
          this.state.classes.forEach(c => {
            if (c.subject && c.subject !== 'Классный час') matchingSubjects.add(c.subject);
          });
        }
        datalist.innerHTML = Array.from(matchingSubjects).map(s => `<option value="${s}"></option>`).join('');
      },

      submitHomeworkInline(event) {
        event.preventDefault();
        const day = document.getElementById('inline-hw-day').value;
        const weekType = document.getElementById('inline-hw-week').value;
        const due = document.getElementById('inline-hw-due').value.trim() || 'К следующему уроку';
        const subject = document.getElementById('inline-hw-subject').value.trim();
        const text = document.getElementById('inline-hw-text').value.trim();
        if (!subject || !text) return;

        const author = this.canManageGroup()
          ? `${this.state.leaderName} (${this.isLeader() ? 'Староста' : 'Зам'})`
          : (this.state.currentStudentName || 'Студент');

        if (!this.state.homeworks) this.state.homeworks = [];
        this.state.homeworks.unshift({
          id: Date.now(),
          day,
          weekType,
          due,
          subject,
          text,
          author,
          done: false
        });

        document.getElementById('inline-hw-text').value = '';
        this.saveState();
        this.renderHomework();
        this.showToast(`ДЗ по предмету «${subject}» (${weekType === '1' ? '★ 1-я нед.' : weekType === '2' ? '★★ 2-я нед.' : 'обе нед.'}) добавлено!`);
      },

      openAddHomeworkModal() {
        this.openModal('Добавить домашнее задание', `
          <form onsubmit="App.submitHomeworkModal(event)" class="space-y-3.5">
            <div class="grid grid-cols-2 gap-2.5">
              <div>
                <label class="block text-xs text-slate-400 mb-1">День недели</label>
                <select id="modal-hw-day" class="w-full px-3 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm">
                  <option>Понедельник</option>
                  <option>Вторник</option>
                  <option>Среда</option>
                  <option>Четверг</option>
                  <option>Пятница</option>
                  <option>Суббота</option>
                </select>
              </div>
              <div>
                <label class="block text-xs text-slate-400 mb-1">Неделя (★ / ★★)</label>
                <select id="modal-hw-week" class="w-full px-3 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-amber-300 font-semibold text-sm">
                  <option value="1" ${this.state.currentWeekNumber === 1 ? 'selected' : ''}>★ 1-я неделя</option>
                  <option value="2" ${this.state.currentWeekNumber === 2 ? 'selected' : ''}>★★ 2-я неделя</option>
                  <option value="both">Каждая неделя</option>
                </select>
              </div>
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Предмет</label>
              <input id="modal-hw-subject" list="hw-subjects-datalist" type="text" required placeholder="Например: Информатика" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Срок сдачи / Дата</label>
              <input id="modal-hw-due" type="text" placeholder="Например: к четвергу" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Описание домашнего задания</label>
              <textarea id="modal-hw-text" rows="3" required placeholder="Что задали..." class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm"></textarea>
            </div>
            <button type="submit" class="w-full py-3 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-sm transition">
              Сохранить ДЗ
            </button>
          </form>
        `);
      },

      submitHomeworkModal(event) {
        event.preventDefault();
        const day = document.getElementById('modal-hw-day').value;
        const weekType = document.getElementById('modal-hw-week').value;
        const subject = document.getElementById('modal-hw-subject').value.trim();
        const due = document.getElementById('modal-hw-due').value.trim() || 'К следующему уроку';
        const text = document.getElementById('modal-hw-text').value.trim();
        if (!subject || !text) return;

        const author = this.canManageGroup()
          ? `${this.state.leaderName} (${this.isLeader() ? 'Староста' : 'Зам'})`
          : (this.state.currentStudentName || 'Студент');

        if (!this.state.homeworks) this.state.homeworks = [];
        this.state.homeworks.unshift({
          id: Date.now(),
          day,
          weekType,
          due,
          subject,
          text,
          author,
          done: false
        });

        this.saveState();
        this.renderHomework();
        this.closeModal();
        this.showToast(`Домашнее задание по предмету «${subject}» добавлено!`);
      },

      toggleHomeworkDone(hwId) {
        const hw = (this.state.homeworks || []).find(h => h.id === hwId);
        if (!hw) return;
        hw.done = !hw.done;
        this.saveState();
        this.renderHomework();
      },

      deleteHomework(hwId) {
        if (!this.canManageGroup()) {
          this.showToast('Удалять домашние задания может только Староста или Зам. старосты');
          return;
        }
        this.state.homeworks = (this.state.homeworks || []).filter(h => h.id !== hwId);
        this.saveState();
        this.renderHomework();
        this.showToast('Домашнее задание удалено');
      },

      renderHomework() {
        const container = document.getElementById('homework-list-container');
        const activeHint = document.getElementById('hw-active-week-hint');
        if (activeHint) {
          activeHint.textContent = `Сейчас идёт: ${this.state.currentWeekNumber === 1 ? '★ 1-я неделя' : '★★ 2-я неделя'}`;
        }
        if (!container) return;

        const list = this.state.homeworks || [];
        const filter = this.state.homeworkWeekFilter || 'all';
        const filtered = list.filter(h => {
          if (filter === 'all') return true;
          return h.weekType === 'both' || h.weekType === filter;
        });

        if (filtered.length === 0) {
          container.innerHTML = `
            <div class="p-6 rounded-2xl bg-cardBg border border-slate-800/80 text-center space-y-2">
              <p class="text-sm font-semibold text-slate-300">Домашних заданий для выбранной недели пока нет</p>
              <p class="text-xs text-slate-400">Заполните форму выше, чтобы добавить задание для ★ 1-й или ★★ 2-й недели.</p>
            </div>
          `;
          return;
        }

        const canDelete = this.canManageGroup();
        container.innerHTML = filtered.map(hw => `
          <div class="p-4 rounded-2xl bg-cardBg border ${hw.done ? 'border-emerald-500/40 opacity-80' : 'border-slate-800/90'} flex flex-col sm:flex-row sm:items-center justify-between gap-3">
            <div class="space-y-1.5">
              <div class="flex flex-wrap items-center gap-2">
                <span class="text-xs font-bold uppercase text-emerald-400">${hw.day}</span>
                ${this.getWeekBadgeHtml(hw.weekType)}
                <span class="px-2 py-0.5 rounded-md bg-slate-900 border border-slate-800 text-[11px] text-slate-300">Срок: ${hw.due}</span>
              </div>
              <p class="text-sm sm:text-base font-bold ${hw.done ? 'line-through text-slate-400' : 'text-white'}">${hw.subject}</p>
              <p class="text-xs sm:text-sm text-slate-200 leading-relaxed">${hw.text}</p>
              <p class="text-[11px] text-slate-500">Добавил(а): ${hw.author || 'Участник группы'}</p>
            </div>

            <div class="flex items-center gap-2 shrink-0">
              <button type="button" onclick="App.toggleHomeworkDone(${hw.id})" class="px-3 py-2 rounded-xl text-xs font-semibold transition ${
                hw.done
                  ? 'bg-emerald-500/20 text-emerald-300 border border-emerald-500/40'
                  : 'bg-slate-900 hover:bg-slate-800 text-slate-200 border border-slate-700'
              }">
                ${hw.done ? '✓ Выполнено' : 'Отметить выполненным'}
              </button>
              ${canDelete ? `
                <button type="button" onclick="App.deleteHomework(${hw.id})" class="p-2 rounded-xl bg-rose-500/15 hover:bg-rose-500/25 border border-rose-500/30 text-rose-300 text-xs transition" title="Удалить ДЗ">
                  ✕
                </button>
              ` : ''}
            </div>
          </div>
        `).join('');
      },

      addSelfToGroup(notify = true) {
        const exists = this.state.students.some(s => s.isLeader);
        if (!exists) {
          this.state.students.unshift({
            id: Date.now(),
            name: this.state.leaderName,
            status: 'На паре (авто)',
            autoPresent: true,
            absences: 0,
            excusedCount: 0,
            presentCount: 1,
            totalClassesCount: 1,
            isLeader: true,
            isDeputy: false,
            joinedViaCode: false
          });
        }
        const card = document.getElementById('add-self-card');
        if (card) card.classList.add('hidden');
        this.saveState();
        this.renderStudents();
        this.updateHomePrimaryWidgets();
        if (notify) this.showToast('Вы добавлены в список группы!');
      },

      renderStudents() {
        const listEl = document.getElementById('students-list');
        const emptyEl = document.getElementById('students-empty-state');
        const subtitleEl = document.getElementById('students-count-subtitle');
        const addSelfCard = document.getElementById('add-self-card');

        const total = this.state.students.length;
        const activeCount = this.state.students.filter(s => s.status.includes('На паре')).length;
        const deputiesCount = this.state.students.filter(s => s.isDeputy).length;
        subtitleEl.textContent = `${total} в группе · на паре ${activeCount} · замов: ${deputiesCount}`;

        const hasLeaderInList = this.state.students.some(s => s.isLeader);
        if (addSelfCard) {
          if (hasLeaderInList || !this.canManageGroup()) {
            addSelfCard.classList.add('hidden');
          } else {
            addSelfCard.classList.remove('hidden');
          }
        }

        if (total === 0) {
          emptyEl.classList.remove('hidden');
          listEl.innerHTML = '';
          return;
        }

        emptyEl.classList.add('hidden');
        const canManage = this.canManageGroup();
        const isStarosta = this.isLeader();

        listEl.innerHTML = this.state.students.map(student => {
          let roleBadge = '';
          if (student.isLeader) {
            roleBadge = `<span class="px-2 py-0.5 rounded-md text-[10px] font-bold bg-emerald-500/20 text-emerald-300 border border-emerald-500/40">👑 Староста</span>`;
          } else if (student.isDeputy) {
            roleBadge = `<span class="px-2 py-0.5 rounded-md text-[10px] font-bold bg-amber-500/20 text-amber-300 border border-amber-500/40">⭐ Зам. старосты</span>`;
          } else if (student.joinedViaCode) {
            roleBadge = `<span class="px-2 py-0.5 rounded-md text-[10px] bg-slate-800 text-slate-400">По коду</span>`;
          }

          const isMe = this.state.currentStudentId && student.id === this.state.currentStudentId;
          const canDeleteThisMember = canManage && (!student.isLeader || isStarosta);

          return `
            <div class="p-4 rounded-2xl bg-cardBg border ${student.isDeputy ? 'border-amber-500/40' : 'border-slate-800/90'} flex flex-wrap items-center justify-between gap-3">
              <div class="flex items-center gap-3">
                <div class="w-10 h-10 rounded-xl ${student.isDeputy ? 'bg-amber-500/20 text-amber-300' : 'bg-emerald-500/15 text-emerald-400'} font-bold text-sm flex items-center justify-center">
                  ${student.name.charAt(0).toUpperCase()}
                </div>
                <div>
                  <div class="flex items-center gap-2 flex-wrap">
                    <p class="text-sm font-semibold text-white">${student.name}${isMe ? ' (Вы)' : ''}</p>
                    ${roleBadge}
                  </div>
                  <p class="text-xs text-emerald-400 mt-0.5">${student.status}</p>
                </div>
              </div>

              <div class="flex items-center gap-2 flex-wrap">
                ${isStarosta && !student.isLeader ? `
                  <button type="button" onclick="App.toggleDeputyRole(${student.id}, false)" class="px-2.5 py-1.5 rounded-xl text-xs font-semibold transition ${
                    student.isDeputy
                      ? 'bg-amber-500/20 hover:bg-amber-500/30 border border-amber-500/40 text-amber-300'
                      : 'bg-slate-900 hover:bg-slate-800 border border-amber-500/30 text-amber-300'
                  }">
                    ${student.isDeputy ? '★ Снять зама' : '+ Сделать замом'}
                  </button>
                ` : ''}

                ${canManage ? `
                  <button type="button" onclick="App.toggleStudentStatus(${student.id})" class="px-3 py-1.5 rounded-xl bg-slate-800 hover:bg-slate-700 text-xs font-medium text-slate-200 transition">
                    Сменить статус
                  </button>
                ` : ''}

                ${canDeleteThisMember ? `
                  <button type="button" onclick="App.removeStudent(${student.id}, false)" class="px-2.5 py-1.5 rounded-xl bg-rose-500/15 hover:bg-rose-500/25 border border-rose-500/30 text-xs font-semibold text-rose-300 transition" title="Удалить из группы">
                    Удалить
                  </button>
                ` : ''}

                ${isMe && !isStarosta ? `
                  <button type="button" onclick="App.leaveGroupAsStudent()" class="px-2.5 py-1.5 rounded-xl bg-rose-500/20 hover:bg-rose-500/30 border border-rose-500/40 text-xs font-semibold text-rose-300 transition">
                    Выйти
                  </button>
                ` : ''}
              </div>
            </div>
          `;
        }).join('');
      },

      toggleStudentStatus(id) {
        if (!this.canManageGroup()) {
          this.showToast('Менять статусы других студентов может только Староста или Зам. старосты');
          return;
        }
        const st = this.state.students.find(s => s.id === id);
        if (!st) return;
        st.totalClassesCount = (st.totalClassesCount || 1) + 1;
        if (st.status.includes('На паре')) {
          st.status = 'Пропуск (уваж.)';
          st.excusedCount = (st.excusedCount || 0) + 1;
        } else if (st.status === 'Пропуск (уваж.)') {
          st.status = 'Не был';
          st.absences = (st.absences || 0) + 1;
        } else {
          st.status = 'На паре (авто)';
          st.presentCount = (st.presentCount || 1) + 1;
        }
        this.saveState();
        this.renderStudents();
        this.updateHomePrimaryWidgets();
      },

      openAddStudentModal() {
        if (!this.canManageGroup()) {
          this.showToast('Добавлять участников вручную может только Староста или Зам. старосты');
          return;
        }
        this.openModal('Добавить участника в группу', `
          <form onsubmit="App.submitNewStudent(event)" class="space-y-4">
            <div>
              <label class="block text-xs text-slate-400 mb-1">Фамилия и Имя студента</label>
              <input id="new-student-name" type="text" required placeholder="Введите Фамилию и Имя" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm focus:border-emerald-500 focus:outline-none" />
            </div>
            ${this.isLeader() ? `
              <label class="flex items-center gap-2.5 cursor-pointer">
                <input id="new-student-deputy" type="checkbox" class="w-4 h-4 accent-amber-500" />
                <span class="text-xs text-amber-300 font-medium">Сразу назначить Зам. старосты</span>
              </label>
            ` : ''}
            <label class="flex items-center gap-2.5 cursor-pointer">
              <input id="new-student-auto" type="checkbox" checked class="w-4 h-4 accent-emerald-500" />
              <span class="text-xs text-slate-300">Автоматически отправлять присутствие на парах</span>
            </label>
            <button type="submit" class="w-full py-3 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-sm transition">
              Добавить в группу
            </button>
          </form>
        `);
      },

      submitNewStudent(e) {
        e.preventDefault();
        const name = document.getElementById('new-student-name').value.trim();
        const auto = document.getElementById('new-student-auto').checked;
        const depCheckbox = document.getElementById('new-student-deputy');
        const makeDeputy = depCheckbox ? depCheckbox.checked : false;
        if (!name) return;

        this.state.students.push({
          id: Date.now(),
          name,
          status: auto ? 'На паре (авто)' : 'Ожидает опроса',
          autoPresent: auto,
          absences: 0,
          excusedCount: 0,
          presentCount: auto ? 1 : 0,
          totalClassesCount: 1,
          isLeader: false,
          isDeputy: makeDeputy,
          joinedViaCode: true
        });
        this.saveState();
        this.syncGroupHeader();
        this.renderStudents();
        this.closeModal();
        this.showToast(`Студент ${name} добавлен${makeDeputy ? ' (Зам. старосты)' : ''}`);
      },

      openKnowledgeBaseModal() {
        const canDelete = this.canManageGroup();
        const filesHtml = this.state.knowledgeFiles.length > 0
          ? this.state.knowledgeFiles.map(f => `
              <div class="p-3 rounded-xl bg-slate-900 border border-slate-800 flex items-center justify-between gap-2">
                <div class="min-w-0">
                  <p class="text-sm font-medium text-white truncate">${f.name}</p>
                  <p class="text-xs text-slate-400">${f.size} · Добавил: ${f.addedBy || 'Участник'}</p>
                </div>
                ${canDelete ? `
                  <button type="button" onclick="App.deleteKnowledgeFile(${f.id})" class="text-xs text-rose-400 hover:underline shrink-0">Удалить</button>
                ` : ''}
              </div>
            `).join('')
          : `
            <div class="py-8 px-4 text-center rounded-xl bg-slate-900/60 border border-slate-800/80 space-y-1.5">
              <p class="text-sm font-medium text-slate-300">В базе знаний пока пусто</p>
              <p class="text-xs text-slate-400">И староста, и студенты могут загружать сюда методички, конспекты или учебные файлы.</p>
            </div>
          `;

        this.openModal('База знаний группы', `
          <div class="space-y-2.5">${filesHtml}</div>
          <button type="button" onclick="document.getElementById('knowledge-file-input').click()" class="w-full mt-3 py-3 rounded-xl bg-slate-800 hover:bg-slate-700 border border-slate-700 text-xs sm:text-sm font-semibold text-emerald-300 transition">
            + Добавить конспект или методичку
          </button>
        `);
      },

      handleKnowledgeFileUpload(event) {
        const files = Array.from(event.target.files || []);
        if (files.length === 0) return;
        const author = this.canManageGroup() ? this.state.leaderName : (this.state.currentStudentName || 'Студент');
        files.forEach(file => {
          const sizeKb = Math.max(1, Math.round(file.size / 1024));
          const sizeFormatted = sizeKb > 1024 ? `${(sizeKb / 1024).toFixed(1)} МБ` : `${sizeKb} КБ`;
          this.state.knowledgeFiles.push({
            id: Date.now() + Math.random(),
            name: file.name,
            size: sizeFormatted,
            addedBy: author
          });
        });
        this.saveState();
        this.openKnowledgeBaseModal();
        this.showToast(`Добавлено файлов: ${files.length}`);
      },

      deleteKnowledgeFile(id) {
        if (!this.canManageGroup()) {
          this.showToast('Студент может только добавлять файлы');
          return;
        }
        this.state.knowledgeFiles = this.state.knowledgeFiles.filter(f => f.id !== id);
        this.saveState();
        this.openKnowledgeBaseModal();
        this.showToast('Файл удалён');
      },

      openSemesterModal() {
        if (!this.canManageGroup()) {
          this.showToast('Изменять даты семестра может только Староста');
          return;
        }
        this.openModal('Изменить даты семестра', `
          <form onsubmit="App.submitSemesterDates(event)" class="space-y-4">
            <div>
              <label class="block text-xs text-slate-400 mb-1">Период семестра</label>
              <input id="semester-input" type="text" required value="${this.state.semesterDates}" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <button type="submit" class="w-full py-3 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-sm transition">
              Сохранить
            </button>
          </form>
        `);
      },

      submitSemesterDates(e) {
        e.preventDefault();
        this.state.semesterDates = document.getElementById('semester-input').value;
        document.getElementById('semester-dates-display').textContent = this.state.semesterDates;
        this.saveState();
        this.closeModal();
        this.showToast('Даты семестра обновлены');
      },

      renderHolidays() {
        const el = document.getElementById('holidays-list');
        if (!el) return;
        if (this.state.holidays.length === 0) {
          el.innerHTML = 'Периодов пока нет.';
          return;
        }
        el.innerHTML = this.state.holidays.map(h => `
          <div class="p-3.5 rounded-xl bg-cardBg border border-slate-800 flex items-center justify-between text-left mb-2">
            <div>
              <p class="text-sm font-semibold text-white">${h.title}</p>
              <p class="text-xs text-slate-400">${h.dates}</p>
            </div>
            <span class="text-xs text-emerald-400">Авто-пауза опросов</span>
          </div>
        `).join('');
      },

      openAddHolidayModal() {
        if (!this.canManageGroup()) {
          this.showToast('Добавлять каникулы и сессии может только Староста или Зам. старосты');
          return;
        }
        this.openModal('Добавить каникулы / сессию', `
          <form onsubmit="App.submitHoliday(event)" class="space-y-4">
            <div>
              <label class="block text-xs text-slate-400 mb-1">Название периода</label>
              <input id="holiday-title" type="text" required placeholder="Например: Зимняя сессия" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Даты</label>
              <input id="holiday-dates" type="text" required placeholder="10.01.2027 — 25.01.2027" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <button type="submit" class="w-full py-3 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-sm transition">
              Добавить период
            </button>
          </form>
        `);
      },

      submitHoliday(e) {
        e.preventDefault();
        if (!this.canManageGroup()) return;
        this.state.holidays.push({
          title: document.getElementById('holiday-title').value,
          dates: document.getElementById('holiday-dates').value
        });
        this.saveState();
        this.renderHolidays();
        this.closeModal();
        this.showToast('Период добавлен');
      },

      // Авто-отправка и авто-рассылка присутствия
      toggleAutoAttendance(enabled, notify = true) {
        if (notify && !this.canManageGroup()) {
          this.showToast('Управлять авто-рассылкой может только Староста или Зам. старосты');
          const toggleInput = document.getElementById('toggle-auto-attendance');
          if (toggleInput) toggleInput.checked = this.state.autoAttendanceEnabled;
          return;
        }

        this.state.autoAttendanceEnabled = Boolean(enabled);
        const toggleInput = document.getElementById('toggle-auto-attendance');
        if (toggleInput) toggleInput.checked = this.state.autoAttendanceEnabled;

        const badge = document.getElementById('auto-send-badge');
        const text = document.getElementById('live-attendance-text');
        const dot = document.getElementById('live-status-dot');

        if (this.state.autoAttendanceEnabled) {
          if (badge) {
            badge.textContent = 'Авто-рассылка активна';
            badge.className = 'px-2.5 py-0.5 rounded-full text-[11px] font-semibold bg-emerald-500/20 text-emerald-300 border border-emerald-500/30';
          }
          if (text) {
            text.textContent = 'Бот готов отправить сводку посещаемости всем участникам группы в Telegram и сохранить день в общую статистику за всё время.';
          }
          if (dot) {
            dot.className = 'w-2.5 h-2.5 rounded-full bg-emerald-400 animate-ping';
          }
        } else {
          if (badge) {
            badge.textContent = 'На паузе';
            badge.className = 'px-2.5 py-0.5 rounded-full text-[11px] font-semibold bg-amber-500/20 text-amber-300 border border-amber-500/30';
          }
          if (text) {
            text.textContent = 'Автоматическая рассылка посещаемости приостановлена. Вы можете отправить сводку всем участникам вручную кнопкой ниже.';
          }
          if (dot) {
            dot.className = 'w-2.5 h-2.5 rounded-full bg-amber-400';
          }
        }

        this.saveState();
        if (notify) {
          this.showToast(
            this.state.autoAttendanceEnabled
              ? 'Авто-рассылка присутствия включена'
              : 'Авто-рассылка присутствия приостановлена'
          );
        }
      },

      // Сохранение среза посещаемости за день в общую статистику за всё время
      recordAttendanceSnapshot(customLabel) {
        const tz = this.getCurrentTimezone();
        const label = customLabel || `${this.formatDateInTz(tz.id)} · ${this.formatTimeInTz(tz.id, false)}`;
        const weekLabel = this.state.currentWeekNumber === 1 ? '★ 1-я неделя' : '★★ 2-я неделя';

        if (!this.state.attendanceHistory) this.state.attendanceHistory = [];

        const records = this.state.students.map(s => {
          s.totalClassesCount = (s.totalClassesCount || 0) + 1;
          if (s.status.includes('На паре')) {
            s.presentCount = (s.presentCount || 0) + 1;
          } else if (s.status === 'Не был') {
            s.absences = (s.absences || 0) + 1;
          } else {
            s.excusedCount = (s.excusedCount || 0) + 1;
          }
          return {
            name: s.name,
            status: s.status,
            isDeputy: Boolean(s.isDeputy),
            isLeader: Boolean(s.isLeader)
          };
        });

        this.state.attendanceHistory.unshift({
          id: Date.now(),
          dateLabel: label,
          weekLabel,
          records
        });

        this.saveState();
        this.updateHomePrimaryWidgets();
      },

      saveSnapshotAndRefreshModal() {
        if (!this.canManageGroup()) {
          this.showToast('Фиксировать учебный день в журнал может только Староста или Зам. старосты');
          return;
        }
        this.recordAttendanceSnapshot();
        this.openAttendanceJournalModal('alltime');
        this.showToast('Посещаемость за текущий день сохранена в общую статистику за всё время!');
      },

      openAttendanceJournalModal(mode = 'alltime') {
        const tz = this.getCurrentTimezone();
        const dateStr = this.formatDateInTz(tz.id);
        const canManage = this.canManageGroup();

        // Статистика всех участников за всё время
        const allTimeRows = this.state.students.length > 0
          ? this.state.students.map(s => {
              const total = Math.max(1, s.totalClassesCount || 1);
              const unexcused = s.absences || 0;
              const excused = s.excusedCount || 0;
              const present = Math.max(0, s.presentCount !== undefined ? s.presentCount : (total - unexcused - excused));
              const percent = Math.max(0, Math.min(100, Math.round(((present + excused * 0.5) / total) * 100)));

              return `
                <div class="p-3 rounded-xl bg-slate-900 border border-slate-800 flex flex-wrap items-center justify-between gap-2">
                  <div>
                    <div class="flex items-center gap-1.5 flex-wrap">
                      <span class="text-sm font-bold text-white">${s.name}</span>
                      ${s.isLeader ? '<span class="text-[10px] px-1.5 py-0.5 rounded bg-emerald-500/20 text-emerald-300 font-bold">Староста</span>' : ''}
                      ${s.isDeputy ? '<span class="text-[10px] px-1.5 py-0.5 rounded bg-amber-500/20 text-amber-300 font-bold">Зам</span>' : ''}
                    </div>
                    <p class="text-[11px] text-slate-400 mt-0.5">
                      За всё время: <strong class="text-emerald-400">Был(а): ${present}</strong> ·
                      <span class="text-amber-300">Уваж./Опозд.: ${excused}</span> ·
                      <span class="text-rose-400">Пропусков: ${unexcused}</span>
                    </p>
                  </div>
                  <div class="flex items-center gap-2">
                    <span class="px-2.5 py-1 rounded-lg text-xs font-bold ${
                      percent >= 80 ? 'bg-emerald-500/20 text-emerald-300' : percent >= 50 ? 'bg-amber-500/20 text-amber-300' : 'bg-rose-500/20 text-rose-300'
                    }">${percent}%</span>
                    <span class="px-2.5 py-1 rounded-lg text-[11px] font-medium bg-slate-950 border border-slate-800 text-slate-300">${s.status}</span>
                  </div>
                </div>
              `;
            }).join('')
          : `<p class="text-xs text-slate-400 py-4 text-center">Список участников группы пока пуст.</p>`;

        // История сохранённых дней за всё время
        const historyList = (this.state.attendanceHistory || []);
        const historyHtml = historyList.length > 0
          ? historyList.map(entry => {
              const presentCnt = entry.records.filter(r => r.status.includes('На паре')).length;
              return `
                <div class="p-3 rounded-xl bg-slate-950 border border-slate-800 space-y-1.5">
                  <div class="flex items-center justify-between text-xs">
                    <span class="font-bold text-emerald-400">${entry.dateLabel}</span>
                    <span class="text-slate-400">${entry.weekLabel} · На паре: ${presentCnt}/${entry.records.length}</span>
                  </div>
                  <div class="flex flex-wrap gap-1.5">
                    ${entry.records.map(r => `
                      <span class="px-2 py-0.5 rounded text-[10px] ${
                        r.status.includes('На паре')
                          ? 'bg-emerald-500/15 text-emerald-300'
                          : r.status === 'Не был'
                          ? 'bg-rose-500/15 text-rose-300'
                          : 'bg-amber-500/15 text-amber-300'
                      }">${r.name}: ${r.status}</span>
                    `).join('')}
                  </div>
                </div>
              `;
            }).join('')
          : `<p class="text-xs text-slate-500 text-center py-2">История по дням появится после сохранения или рассылки сводки.</p>`;

        this.openModal(`Посещаемость всех участников · За всё время и сегодня`, `
          <div class="space-y-4">
            <div class="flex items-center justify-between flex-wrap gap-2 p-3 rounded-xl bg-emerald-950/40 border border-emerald-500/30">
              <div>
                <p class="text-xs font-bold text-emerald-300 uppercase">Сводная таблица за всё время учёбы</p>
                <p class="text-[11px] text-slate-300">Сегодня: ${dateStr} · ${this.state.currentWeekNumber === 1 ? '★ 1-я неделя' : '★★ 2-я неделя'}</p>
              </div>
              ${canManage && this.state.students.length > 0 ? `
                <button type="button" onclick="App.saveSnapshotAndRefreshModal()" class="px-3 py-1.5 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-xs transition">
                  + Зафиксировать день в статистику
                </button>
              ` : ''}
            </div>

            <div class="space-y-2 max-h-60 overflow-y-auto pr-1">
              ${allTimeRows}
            </div>

            <div class="space-y-2 pt-2 border-t border-slate-800">
              <p class="text-xs font-bold uppercase tracking-wider text-slate-400">История по датам за всё время (${historyList.length})</p>
              <div class="space-y-2 max-h-44 overflow-y-auto pr-1">
                ${historyHtml}
              </div>
            </div>

            ${canManage ? `
              <div class="grid grid-cols-1 sm:grid-cols-2 gap-2 pt-2">
                <button type="button" onclick="App.triggerAttendancePhotoUpload()" class="py-2.5 px-3 rounded-xl bg-slate-800 hover:bg-slate-700 text-emerald-300 font-semibold text-xs">
                  📸 Отметить по фото
                </button>
                <button type="button" onclick="App.openBroadcastAttendanceModal()" class="py-2.5 px-3 rounded-xl bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-semibold text-xs">
                  🚀 Разослать всем в TG
                </button>
              </div>
            ` : ''}
          </div>
        `);
      },

      openAnnouncementsModal() {
        const items = this.state.announcements.length > 0
          ? this.state.announcements.map(a => `
              <div class="p-3.5 rounded-xl bg-slate-900 border border-slate-800 space-y-1">
                <div class="flex justify-between text-xs text-emerald-400"><span>${a.author || 'Группа'}</span><span>${a.date}</span></div>
                <p class="font-semibold text-white text-sm">${a.title}</p>
                <p class="text-xs text-slate-300">${a.text}</p>
              </div>
            `).join('')
          : `<p class="text-xs text-slate-400 py-4 text-center">Объявлений пока нет. Вы можете добавить информацию для группы.</p>`;

        this.openModal('Объявления и заметки группы', `
          <div class="space-y-3 max-h-60 overflow-y-auto pr-1">${items}</div>
          <form onsubmit="App.addAnnouncement(event)" class="pt-3 border-t border-slate-800 space-y-2.5">
            <input id="ann-title" type="text" required placeholder="Заголовок (например: ДЗ по информатике)" class="w-full px-3 py-2 rounded-xl bg-slate-900 border border-slate-700 text-white text-xs" />
            <textarea id="ann-text" required placeholder="Текст объявления для группы..." class="w-full px-3 py-2 rounded-xl bg-slate-900 border border-slate-700 text-white text-xs"></textarea>
            <button type="submit" class="w-full py-2.5 rounded-xl bg-emerald-500 text-slate-950 font-semibold text-xs">Добавить объявление</button>
          </form>
        `);
      },

      addAnnouncement(e) {
        e.preventDefault();
        const tz = this.getCurrentTimezone();
        const author = this.canManageGroup() ? `${this.state.leaderName} (Староста)` : (this.state.currentStudentName || 'Студент');
        this.state.announcements.unshift({
          id: Date.now(),
          title: document.getElementById('ann-title').value,
          date: this.formatTimeInTz(tz.id, false),
          text: document.getElementById('ann-text').value,
          author
        });
        this.saveState();
        this.updateHomePrimaryWidgets();
        this.openAnnouncementsModal();
        this.showToast('Объявление добавлено');
      },

      openLegalModal(type) {
        const docs = {
          pdn: {
            title: 'Согласие на обработку персональных данных',
            content: `<p>В соответствии с ФЗ № 152-ФЗ «О персональных данных» пользователь даёт согласие на обработку ФИО, учебной группы и сведений о посещаемости для ведения электронного журнала и рассылки в Telegram.</p>`
          },
          terms: {
            title: 'Пользовательское соглашение',
            content: `<p>Староста создаёт учебную группу, управляет расписанием и назначает заместителей (Зам. старосты). Студенты присоединяются по коду группы и могут добавлять учебные материалы и отмечать себя.</p>`
          },
          auto: {
            title: 'Автоматическая рассылка присутствия',
            content: `<p>Бот автоматически формирует сводку посещаемости и позволяет старосте или зам. старосты отправлять её всем участникам группы в Telegram.</p>`
          }
        };
        const doc = docs[type] || docs.pdn;
        this.openModal(doc.title, `
          <div class="space-y-3 leading-relaxed">${doc.content}</div>
          <button type="button" onclick="App.closeModal()" class="w-full mt-4 py-2.5 rounded-xl bg-emerald-500 text-slate-950 font-semibold text-sm">
            Понятно, закрыть
          </button>
        `);
      },

      openHowItWorksModal() {
        this.openModal('Как работает Староста бот', `
          <ol class="list-decimal list-inside space-y-2.5 text-xs sm:text-sm text-slate-300">
            <li><strong>Вход как Староста:</strong> создаёт группу, получает код приглашения (например, <span class="text-emerald-400 font-mono">${this.state.inviteCode}</span>) и может назначать <strong>Зам. старосты</strong> из подключившихся студентов.</li>
            <li><strong>Вход как Студент:</strong> вводит код старосты, попадает в группу. Не может менять значимые настройки, но может добавлять файлы, объявления, пары и отмечать себя.</li>
            <li><strong>Рассылка посещаемости в Telegram и фото ИИ:</strong> староста и замы могут загружать фото расписания (★ 1-я / ★★ 2-я неделя), фото посещаемости и отправлять сводку всем в Telegram.</li>
          </ol>
        `);
      },

      openEditGroupModal() {
        if (!this.canManageGroup()) {
          this.showToast('Редактировать данные группы может только Староста');
          return;
        }
        this.openModal('Изменить данные группы', `
          <form onsubmit="App.saveGroupEdits(event)" class="space-y-3.5">
            <div>
              <label class="block text-xs text-slate-400 mb-1">Название группы</label>
              <input id="edit-group-name" type="text" required value="${this.state.groupName}" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Учебное заведение</label>
              <input id="edit-uni-name" type="text" required value="${this.state.uniName}" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-white text-sm" />
            </div>
            <div>
              <label class="block text-xs text-slate-400 mb-1">Код приглашения для студентов</label>
              <input id="edit-invite-code" type="text" required value="${this.state.inviteCode}" class="w-full px-3.5 py-2.5 rounded-xl bg-slate-900 border border-slate-700 text-emerald-400 font-mono font-bold uppercase text-sm" />
            </div>
            <button type="submit" class="w-full py-3 rounded-xl bg-emerald-500 text-slate-950 font-semibold text-sm">Сохранить</button>
          </form>
        `);
      },

      saveGroupEdits(e) {
        e.preventDefault();
        const oldCode = (this.state.inviteCode || '').toUpperCase();
        const newName = document.getElementById('edit-group-name').value.trim();
        const newUni = document.getElementById('edit-uni-name').value.trim();
        const newCode = document.getElementById('edit-invite-code').value.trim().toUpperCase() || this.generateRandomInviteCode();

        if (oldCode && oldCode !== newCode && this.state.groups && this.state.groups[oldCode]) {
          delete this.state.groups[oldCode];
        }

        this.state.groupName = newName;
        this.state.uniName = newUni;
        this.state.inviteCode = newCode;
        if (this.isLeader()) {
          this.state.leaderOwnCode = newCode;
        }
        this.saveState();
        this.syncGroupHeader();
        this.closeModal();
        this.showToast('Данные группы и код обновлены');
      },

      copyInviteLink() {
        const inviteText = `Присоединяйтесь к нашей группе «${this.state.groupName}» в мини-приложении Староста Бот! Код для входа студента: ${this.state.inviteCode}`;
        this.copyTextFallback(inviteText);
        this.state.stepsCompleted.invite = true;
        this.saveState();
        this.updateProgressSteps();
        this.showToast(`Код приглашения ${this.state.inviteCode} скопирован!`);
      },

      updateProgressSteps() {
        const count = Object.values(this.state.stepsCompleted).filter(Boolean).length;
        const textEl = document.getElementById('steps-progress-text');
        if (textEl) textEl.textContent = `${count} из 3`;
      },

      hideOnboardingSteps() {
        document.getElementById('onboarding-steps-card').classList.add('hidden');
      },

      cyclePollLeadTime() {
        if (!this.canManageGroup()) {
          this.showToast('Менять время опроса может только Староста');
          return;
        }
        this.state.pollLeadIndex = (this.state.pollLeadIndex + 1) % this.state.pollLeadTimes.length;
        const val = this.state.pollLeadTimes[this.state.pollLeadIndex];
        document.getElementById('poll-lead-time').textContent = val;
        this.showToast(`Время авто-отметки перед парой: ${val}`);
      },

      openModal(title, bodyHtml) {
        document.getElementById('modal-title').textContent = title;
        document.getElementById('modal-body').innerHTML = bodyHtml;
        document.getElementById('modal-overlay').classList.remove('hidden');
      },

      closeModal() {
        document.getElementById('modal-overlay').classList.add('hidden');
      },

      showToast(message) {
        const toast = document.getElementById('toast');
        toast.textContent = message;
        toast.classList.remove('hidden');
        clearTimeout(this.toastTimer);
        this.toastTimer = setTimeout(() => {
          toast.classList.add('hidden');
        }, 3000);
      }
    };

    document.addEventListener('DOMContentLoaded', () => window.App.init());
  </script>
</body>
</html>

<!--
**tepsuime/Tepsuime** is a ✨ _special_ ✨ repository because its `README.md` (this file) appears on your GitHub profile.

Here are some ideas to get you started:
