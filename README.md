#!/bin/bash

# Установщик для приложения шифрования (Атбаш + Гронсфельд)

# --- Конфигурация ---
APP_NAME="encryption-rgr"               # Имя приложения
INSTALL_DIR="/opt/${APP_NAME}"          # Основной каталог установки
BIN_DIR="/usr/local/bin"                # Каталог для глобальных команд
MAIN_EXECUTABLE="main_app"              # Имя скомпилированного приложения
LAUNCHER_NAME="encryption-rgr"          # Команда для запуска из терминала

# Цвета для вывода (для улучшения читаемости)
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Немедленно завершать скрипт при ошибке любой команды
set -e

# --- Функция проверки необходимых инструментов ---
check_dependencies() {
    echo -e "${YELLOW}Проверка зависимостей...${NC}"
    
    # Проверяем наличие компилятора g++
    if ! command -v g++ &> /dev/null; then
        echo -e "${YELLOW}g++ не найден. Установка build-essential...${NC}"
        sudo apt update
        sudo apt install -y build-essential
    else
        echo -e "${GREEN}✓ g++ установлен${NC}"
    fi
    
    # Проверяем наличие make (хотя он не обязателен, но часто нужен)
    if ! command -v make &> /dev/null; then
        echo -e "${YELLOW}make не найден. Установка...${NC}"
        sudo apt install -y make
    else
        echo -e "${GREEN}✓ make установлен${NC}"
    fi
}

# --- Функция компиляции ---
compile_project() {
    echo -e "${YELLOW}Компиляция проекта...${NC}"
    
    # Создаём директорию для библиотек, если её нет
    mkdir -p libs
    
    # 1. Компиляция библиотеки Атбаш
    echo "Компиляция libs/atbash.so..."
    g++ -shared -fPIC -o libs/atbash.so libs/atbash/atbash.cpp -I./
    
    # 2. Компиляция библиотеки Гронсфельда
    echo "Компиляция libs/gronsfeld.so..."
    g++ -shared -fPIC -o libs/gronsfeld.so libs/gronsfeld/gronsfeld.cpp -I./
    
    # 3. Компиляция основного приложения (динамическая загрузка библиотек)
    echo "Компиляция ${MAIN_EXECUTABLE}..."
    g++ -std=c++17 main.cpp -o ${MAIN_EXECUTABLE} -ldl
    
    echo -e "${GREEN}✓ Компиляция завершена успешно${NC}"
}

# --- Функция установки ---
install() {
    echo -e "${GREEN}Начинается установка ${APP_NAME}...${NC}"
    
    # Шаг 0: проверяем зависимости
    check_dependencies
    
    # Шаг 1: компиляция (если исполняемый файл отсутствует)
    if [ ! -f "${MAIN_EXECUTABLE}" ]; then
        compile_project
    else
        echo -e "${YELLOW}Исполняемый файл уже существует. Для пересборки удалите его вручную.${NC}"
    fi
    
    # Шаг 2: создание каталога приложения и копирование файлов
    echo -e "${YELLOW}Копирование файлов в ${INSTALL_DIR}...${NC}"
    sudo mkdir -p "${INSTALL_DIR}/libs"
    sudo cp "${MAIN_EXECUTABLE}" "${INSTALL_DIR}/"
    sudo cp libs/*.so "${INSTALL_DIR}/libs/"
    
    # Шаг 3: создание скрипта-запускатора
    echo -e "${YELLOW}Создание глобальной команды ${LAUNCHER_NAME}...${NC}"
    # Используем tee для записи с правами sudo
    sudo tee "${BIN_DIR}/${LAUNCHER_NAME}" > /dev/null << 'EOF'
#!/bin/bash
# Запускатор для приложения шифрования (Атбаш + Гронсфельд)
# Переходит в каталог установки и запускает основную программу
cd "/opt/encryption-rgr"
./main_app "$@"
EOF
    
    # Шаг 4: установка прав на исполнение
    sudo chmod +x "${BIN_DIR}/${LAUNCHER_NAME}"
    
    # Финальное сообщение
    echo -e "\n${GREEN}Установка успешно завершена!${NC}"
    echo -e "Теперь вы можете запустить приложение командой: ${YELLOW}${LAUNCHER_NAME}${NC}"
    echo -e "Пример: ${LAUNCHER_NAME} --help"
}

# --- Функция удаления ---
uninstall() {
    echo -e "${RED}Удаление ${APP_NAME}...${NC}"
    
    # Запрашиваем подтверждение
    read -p "Вы уверены, что хотите удалить приложение? [y/N] " confirm
    if [[ ! "$confirm" =~ ^[yY](es)?$ ]]; then
        echo "Удаление отменено."
        exit 0
    fi
    
    # Удаляем глобальную команду
    echo "Удаление ${BIN_DIR}/${LAUNCHER_NAME}..."
    sudo rm -f "${BIN_DIR}/${LAUNCHER_NAME}"
    
    # Удаляем каталог приложения
    echo "Удаление ${INSTALL_DIR}..."
    sudo rm -rf "${INSTALL_DIR}"
    
    echo -e "${GREEN}${APP_NAME} полностью удалён из системы.${NC}"
}

# --- Отображение справки ---
show_help() {
    cat << EOF
Использование: sudo ./setup.sh {install|uninstall}

Команды:
  install   - скомпилировать и установить приложение в систему
  uninstall - полностью удалить приложение из системы

При установке будут выполнены следующие действия:
  • Проверка и установка компилятора g++ (через apt)
  • Компиляция динамических библиотек (atbash.so, gronsfeld.so)
  • Компиляция основного приложения (main_app)
  • Копирование файлов в ${INSTALL_DIR}
  • Создание глобальной команды ${LAUNCHER_NAME} в ${BIN_DIR}
EOF
}

# --- Главная логика ---
# Скрипт должен запускаться с sudo (требуются права на запись в /opt и /usr/local/bin)
if [ "$(id -u)" -ne 0 ]; then
    echo -e "${RED}Ошибка: этот скрипт необходимо запускать с правами суперпользователя.${NC}"
    echo "Пожалуйста, используйте: sudo ./setup.sh {install|uninstall}"
    exit 1
fi

# Обработка аргументов командной строки
case "$1" in
    install)
        install
        ;;
    uninstall)
        uninstall
        ;;
    help|--help|-h)
        show_help
        ;;
    *)
        echo -e "${RED}Неизвестная команда: $1${NC}"
        show_help
        exit 1
        ;;
esac

exit 0
