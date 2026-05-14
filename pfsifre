#!/bin/bash

# ─────────────────────────────────────────────
#  psifre — Şifreli Parola Yöneticisi
# ─────────────────────────────────────────────

RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
CYAN='\033[0;36m'
BOLD='\033[1m'
DIM='\033[2m'
NC='\033[0m'

STORE_DIR="${PSIFRE_DIR:-$HOME/.psifre}"
VERIFY_FILE="$STORE_DIR/.verify"
VERIFY_TOKEN="psifre-v1-ok"
BACKUP_MAGIC="PSIFRE-BACKUP-V1"
MASTER_PASS=""
SELECTED_LABEL=""

# ── UI ────────────────────────────────────────

banner() {
  echo -e "${CYAN}${BOLD}"
  echo "  ██████╗ ███████╗██╗███████╗██████╗ ███████╗"
  echo "  ██╔══██╗██╔════╝██║██╔════╝██╔══██╗██╔════╝"
  echo "  ██████╔╝███████╗██║█████╗  ██████╔╝█████╗  "
  echo "  ██╔═══╝ ╚════██║██║██╔══╝  ██╔══██╗██╔══╝  "
  echo "  ██║     ███████║██║██║     ██║  ██║███████╗"
  echo "  ╚═╝     ╚══════╝╚═╝╚═╝     ╚═╝  ╚═╝╚══════╝"
  echo -e "${NC}"
  echo -e "  ${DIM}Şifreli Parola Yöneticisi${NC}"
  echo
}

info() { echo -e "  ${BLUE}→${NC} $*"; }
success() { echo -e "  ${GREEN}✓${NC} $*"; }
warn() { echo -e "  ${YELLOW}!${NC} $*"; }
err() { echo -e "  ${RED}✗${NC} $*" >&2; }
hr() { echo -e "  ${DIM}──────────────────────────────────────────${NC}"; }

ask() {
  echo -en "  ${CYAN}?${NC} ${1}: "
  read -r "$2"
}

ask_secret() {
  echo -en "  ${CYAN}?${NC} ${1}: "
  read -rs "$2"
  echo
}

confirm() {
  echo -en "  ${YELLOW}?${NC} ${1} [e/H]: "
  read -r _ans
  [[ "$_ans" =~ ^[EeYy]$ ]]
}

# ── Pano ──────────────────────────────────────

copy_to_clipboard() {
  if command -v wl-copy &>/dev/null; then
    echo -n "$1" | wl-copy 2>/dev/null && return 0
  elif command -v xclip &>/dev/null; then
    echo -n "$1" | xclip -selection clipboard 2>/dev/null && return 0
  elif command -v xsel &>/dev/null; then
    echo -n "$1" | xsel --clipboard --input 2>/dev/null && return 0
  fi
  return 1
}

# ── Depo ──────────────────────────────────────

init_store() {
  mkdir -p "$STORE_DIR"
  chmod 700 "$STORE_DIR"
}

# ── Ana parola ────────────────────────────────
# Güvenlik: VERIFY_FILE içinde hash saklanmaz.
# Bilinen bir token, ana parola ile AES-256-CBC+PBKDF2
# kullanılarak şifrelenir. Doğrulama sırasında
# şifre çözme başarılı olursa parola doğrudur.

_encrypt_verify() {
  local pass="$1"
  echo -n "$VERIFY_TOKEN" | openssl enc -aes-256-cbc -pbkdf2 -iter 200000 \
    -pass fd:3 -base64 -A 2>/dev/null 3<<<"$pass"
}

_decrypt_verify() {
  local pass="$1"
  echo "$2" | openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 \
    -pass fd:3 -base64 -A 2>/dev/null 3<<<"$pass"
}

setup_master() {
  echo
  echo -e "  ${BOLD}İlk kurulum — Ana parola oluştur${NC}"
  hr
  info "Bu parola tüm kayıtlı parolaları şifreler."
  info "Kaybolursa kayıtlı parolalarınız kurtarılamaz."
  echo

  local pass1 pass2
  ask_secret "Ana parola" pass1
  ask_secret "Ana parola (tekrar)" pass2

  if [[ "$pass1" != "$pass2" ]]; then
    err "Parolalar eşleşmiyor."
    exit 1
  fi

  if [[ ${#pass1} -lt 8 ]]; then
    err "Ana parola en az 8 karakter olmalı."
    exit 1
  fi

  local token
  token=$(_encrypt_verify "$pass1")
  if [[ -z "$token" ]]; then
    err "Şifreleme başarısız (openssl kurulu mu?)."
    exit 1
  fi

  echo "$token" >"$VERIFY_FILE"
  chmod 600 "$VERIFY_FILE"
  success "Ana parola oluşturuldu."
  MASTER_PASS="$pass1"
}

verify_master() {
  local pass
  ask_secret "Ana parola" pass

  local stored decrypted
  stored=$(cat "$VERIFY_FILE")
  decrypted=$(_decrypt_verify "$pass" "$stored")

  if [[ "$decrypted" != "$VERIFY_TOKEN" ]]; then
    err "Hatalı ana parola."
    exit 1
  fi

  MASTER_PASS="$pass"
}

# ── Şifreleme ─────────────────────────────────

encrypt_data() {
  echo -n "$1" | openssl enc -aes-256-cbc -pbkdf2 -iter 200000 \
    -pass fd:3 -base64 -A 2>/dev/null 3<<<"$MASTER_PASS"
}

decrypt_data() {
  echo "$1" | openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 \
    -pass fd:3 -base64 -A 2>/dev/null 3<<<"$MASTER_PASS"
}

# ── Parola üreteci ────────────────────────────

gen_password() {
  local length="${1:-16}" charset="${2:-A-Za-z0-9}"
  cat /dev/urandom | tr -dc "$charset" | head -c "$length"
}

password_strength() {
  local pw="$1" score=0 tips=()
  [[ ${#pw} -ge 12 ]] && ((score++)) || tips+=("uzunluk≥12")
  [[ "$pw" =~ [a-z] ]] && ((score++)) || tips+=("küçük harf")
  [[ "$pw" =~ [A-Z] ]] && ((score++)) || tips+=("büyük harf")
  [[ "$pw" =~ [0-9] ]] && ((score++)) || tips+=("rakam")
  [[ "$pw" =~ [^a-zA-Z0-9] ]] && ((score++)) || tips+=("sembol")

  local tip_str=""
  [[ ${#tips[@]} -gt 0 ]] && tip_str="  ${DIM}(eksik: ${tips[*]})${NC}"

  case $score in
  5) echo -e "${GREEN}Çok güçlü${NC}" ;;
  4) echo -e "${GREEN}Güçlü${NC}${tip_str}" ;;
  3) echo -e "${YELLOW}Orta${NC}${tip_str}" ;;
  2) echo -e "${YELLOW}Zayıf${NC}${tip_str}" ;;
  *) echo -e "${RED}Çok zayıf${NC}${tip_str}" ;;
  esac
}

# ── Kayıt G/Ç ────────────────────────────────

validate_label() {
  local label="$1"
  if [[ ${#label} -gt 64 ]]; then
    err "Etiket en fazla 64 karakter olabilir."
    return 1
  fi
  if [[ "$label" =~ [^a-zA-Z0-9\ \-\._@] ]]; then
    err "Etiket yalnızca harf, rakam, boşluk ve - . _ @ içerebilir."
    return 1
  fi
  if [[ "$label" =~ \.\. ]] || [[ "$label" =~ ^[\ \-] ]]; then
    err "Geçersiz etiket formatı."
    return 1
  fi
}

entry_file() { echo "$STORE_DIR/${1// /_}.enc"; }

save_entry() {
  local label="$1" password="$2" url="${3:-}" note="${4:-}"
  local file payload encrypted

  file=$(entry_file "$label")
  payload=$(printf "label=%s\npassword=%s\nurl=%s\nnote=%s\ndate=%s" \
    "$label" "$password" "$url" "$note" "$(date '+%Y-%m-%d %H:%M')")

  encrypted=$(encrypt_data "$payload")
  if [[ -z "$encrypted" ]]; then
    err "Şifreleme başarısız."
    return 1
  fi

  echo "$encrypted" >"$file"
  chmod 600 "$file"
}

load_entry() {
  local label="$1"
  local file raw decrypted

  file=$(entry_file "$label")
  if [[ ! -f "$file" ]]; then
    err "Kayıt bulunamadı: '$label'"
    return 1
  fi

  raw=$(cat "$file")
  decrypted=$(decrypt_data "$raw")
  if [[ -z "$decrypted" ]]; then
    err "Şifre çözme başarısız."
    return 1
  fi

  echo "$decrypted"
}

list_labels() {
  local f
  for f in "$STORE_DIR"/*.enc; do
    [[ -f "$f" ]] && basename "$f" .enc | tr '_' ' '
  done
}

get_field() { echo "$1" | grep "^${2}=" | cut -d= -f2-; }

# ── Kayıt seçici (global değişken kullanır) ───
# NEDEN: $(resolve_label) gibi komut ikamesi kullanılırsa
# fonksiyon içindeki echo'lar ekranda görünmez — hepsi
# yakalanır. Global SELECTED_LABEL ile bu sorun ortadan kalkar.

select_entry() {
  local prompt_msg="$1"
  local labels=()
  SELECTED_LABEL=""

  while IFS= read -r l; do
    labels+=("$l")
  done < <(list_labels)

  if [[ ${#labels[@]} -eq 0 ]]; then
    warn "Henüz kayıt yok."
    return 1
  fi

  local i=1
  for label in "${labels[@]}"; do
    echo -e "  ${DIM}${i})${NC} ${label}"
    ((i++))
  done
  echo

  local _sel
  ask "$prompt_msg" _sel

  if [[ "$_sel" =~ ^[0-9]+$ ]]; then
    local idx=$((_sel - 1))
    if [[ $idx -ge 0 && $idx -lt ${#labels[@]} ]]; then
      SELECTED_LABEL="${labels[$idx]}"
    else
      err "Geçersiz numara."
      return 1
    fi
  else
    SELECTED_LABEL="$_sel"
  fi
}

# ── Eylemler ──────────────────────────────────

do_generate() {
  echo
  echo -e "  ${BOLD}Parola Üret & Kaydet${NC}"
  hr

  local label
  ask "Etiket (örn: github, e-posta)" label
  label=$(echo "$label" | xargs)
  [[ -z "$label" ]] && {
    err "Etiket boş olamaz."
    return
  }
  validate_label "$label" || return

  local file
  file=$(entry_file "$label")
  if [[ -f "$file" ]]; then
    confirm "'${label}' zaten var. Üzerine yazılsın mı?" || return
  fi

  local len
  ask "Uzunluk [16]" len
  len=${len:-16}
  if ! [[ "$len" =~ ^[0-9]+$ ]] || [[ "$len" -lt 4 ]]; then
    err "Geçersiz uzunluk (en az 4)."
    return
  fi

  echo
  echo -e "  ${BOLD}Karakter seti:${NC}"
  echo -e "  ${DIM}1)${NC} Harf + Rakam (varsayılan)"
  echo -e "  ${DIM}2)${NC} Harf + Rakam + Sembol"
  echo -e "  ${DIM}3)${NC} Sadece rakam"
  echo -e "  ${DIM}4)${NC} Özel"
  local cs_choice
  ask "Seçim [1]" cs_choice

  local charset
  case "${cs_choice:-1}" in
  1) charset="A-Za-z0-9" ;;
  2) charset='A-Za-z0-9!@#$%^&*()-_=+[]{}|;:,.?' ;;
  3) charset="0-9" ;;
  4) ask "Kullanılacak karakterler" charset ;;
  *) charset="A-Za-z0-9" ;;
  esac

  local password
  password=$(gen_password "$len" "$charset")

  echo
  echo -e "  ${BOLD}Üretilen: ${GREEN}${BOLD}${password}${NC}"
  echo -e "  ${BOLD}Güç:      $(password_strength "$password")"
  echo

  local url note
  ask "URL / site (opsiyonel)" url
  ask "Not (opsiyonel)" note

  save_entry "$label" "$password" "$url" "$note" || return

  echo
  success "Kaydedildi: '${label}'"

  if copy_to_clipboard "$password"; then
    success "Panoya kopyalandı."
  fi
}

do_import() {
  echo
  echo -e "  ${BOLD}Elle Parola Kaydet${NC}"
  hr

  local label
  ask "Etiket" label
  label=$(echo "$label" | xargs)
  [[ -z "$label" ]] && {
    err "Etiket boş olamaz."
    return
  }
  validate_label "$label" || return

  local file
  file=$(entry_file "$label")
  if [[ -f "$file" ]]; then
    confirm "'${label}' zaten var. Üzerine yazılsın mı?" || return
  fi

  local password
  ask_secret "Parola" password
  [[ -z "$password" ]] && {
    err "Parola boş olamaz."
    return
  }

  local url note
  ask "URL / site (opsiyonel)" url
  ask "Not (opsiyonel)" note

  save_entry "$label" "$password" "$url" "$note" || return
  success "Kaydedildi ve şifrelendi: '${label}'"
}

do_list() {
  echo
  echo -e "  ${BOLD}Kayıtlı Parolalar${NC}"
  hr

  local labels=() count=0
  while IFS= read -r l; do
    labels+=("$l")
    ((count++))
  done < <(list_labels)

  if [[ $count -eq 0 ]]; then
    warn "Henüz kayıt yok."
    return
  fi

  local i=1
  for label in "${labels[@]}"; do
    echo -e "  ${DIM}${i})${NC} ${label}"
    ((i++))
  done

  echo
  info "Toplam: ${count} kayıt"
}

do_show() {
  echo
  echo -e "  ${BOLD}Parola Göster${NC}"
  hr

  select_entry "Etiket adı veya numara" || return
  local label="$SELECTED_LABEL"

  local raw
  raw=$(load_entry "$label") || return

  local pw url note date
  pw=$(get_field "$raw" "password")
  url=$(get_field "$raw" "url")
  note=$(get_field "$raw" "note")
  date=$(get_field "$raw" "date")

  echo
  echo -e "  ${BOLD}Kayıt:   ${CYAN}${label}${NC}"
  hr
  echo -e "  ${BOLD}Parola:  ${GREEN}${BOLD}${pw}${NC}"
  echo -e "  ${BOLD}Güç:     $(password_strength "$pw")"
  [[ -n "$url" ]] && echo -e "  ${BOLD}URL:     ${NC}${url}"
  [[ -n "$note" ]] && echo -e "  ${BOLD}Not:     ${NC}${note}"
  [[ -n "$date" ]] && echo -e "  ${BOLD}Tarih:   ${DIM}${date}${NC}"
  echo

  if copy_to_clipboard "$pw"; then
    success "Panoya kopyalandı."
  fi
}

do_delete() {
  echo
  echo -e "  ${BOLD}Kayıt Sil${NC}"
  hr

  select_entry "Silmek istediğiniz etiket veya numara" || return
  local label="$SELECTED_LABEL"

  local file
  file=$(entry_file "$label")
  if [[ ! -f "$file" ]]; then
    err "Kayıt bulunamadı."
    return
  fi

  confirm "'${label}' kalıcı olarak silinsin mi?" || {
    warn "İptal edildi."
    return
  }
  rm -f "$file"
  success "'${label}' silindi."
}

do_change_master() {
  echo
  echo -e "  ${BOLD}Ana Parolayı Değiştir${NC}"
  hr
  warn "Tüm kayıtlar yeni parola ile yeniden şifrelenecek."
  echo

  local labels=()
  while IFS= read -r l; do labels+=("$l"); done < <(list_labels)

  # Tüm kayıtları mevcut anahtarla çöz — değişiklikten önce
  declare -A raw_map
  for label in "${labels[@]}"; do
    local raw
    raw=$(load_entry "$label") || {
      err "'$label' okunamadı. İptal."
      return
    }
    raw_map["$label"]="$raw"
  done

  local new1 new2
  ask_secret "Yeni ana parola" new1
  ask_secret "Yeni ana parola (tekrar)" new2

  if [[ "$new1" != "$new2" ]]; then
    err "Parolalar eşleşmiyor."
    return
  fi

  if [[ ${#new1} -lt 8 ]]; then
    err "En az 8 karakter gerekli."
    return
  fi

  # Önce tümünü geçici .new dosyalarına şifrele; hepsi başarılıysa yer değiştir
  local tmp_pairs=() failed=0
  for label in "${!raw_map[@]}"; do
    local raw="${raw_map[$label]}"
    local pw url note
    pw=$(get_field "$raw" "password")
    url=$(get_field "$raw" "url")
    note=$(get_field "$raw" "note")

    local file tmp payload encrypted
    file=$(entry_file "$label")
    tmp="${file}.new"
    payload=$(printf "label=%s\npassword=%s\nurl=%s\nnote=%s\ndate=%s" \
      "$label" "$pw" "$url" "$note" "$(date '+%Y-%m-%d %H:%M')")
    encrypted=$(echo -n "$payload" | openssl enc -aes-256-cbc -pbkdf2 -iter 200000 \
      -pass fd:3 -base64 -A 2>/dev/null 3<<<"$new1")

    if [[ -z "$encrypted" ]]; then
      failed=1
      break
    fi
    echo "$encrypted" >"$tmp"
    chmod 600 "$tmp"
    tmp_pairs+=("$tmp" "$file")
  done

  if [[ $failed -eq 1 ]]; then
    for f in "${tmp_pairs[@]}"; do [[ "$f" == *.new ]] && rm -f "$f"; done
    err "Şifreleme başarısız, hiçbir değişiklik uygulanmadı."
    return
  fi

  # Tüm geçici dosyalar hazır — şimdi atomik olarak uygula
  local token
  token=$(_encrypt_verify "$new1")
  echo "$token" >"$VERIFY_FILE"
  chmod 600 "$VERIFY_FILE"

  local i=0
  while [[ $i -lt ${#tmp_pairs[@]} ]]; do
    mv "${tmp_pairs[$i]}" "${tmp_pairs[$((i + 1))]}"
    ((i += 2))
  done

  MASTER_PASS="$new1"
  success "Ana parola değiştirildi. Tüm kayıtlar yeniden şifrelendi."
}

# ── Yedek ────────────────────────────────────

do_backup() {
  echo
  echo -e "  ${BOLD}Yedek Al${NC}"
  hr

  local labels=()
  while IFS= read -r l; do labels+=("$l"); done < <(list_labels)

  if [[ ${#labels[@]} -eq 0 ]]; then
    warn "Yedeklenecek kayıt yok."
    return
  fi

  local default_out="$HOME/psifre-yedek-$(date '+%Y%m%d-%H%M%S').bak"
  ask "Kayıt yeri [${default_out}]" out_path
  out_path="${out_path:-$default_out}"

  # Her girişi base64 ile kodla (binary-safe), satır satır birleştir
  local backup_plain=""
  for label in "${labels[@]}"; do
    local raw
    raw=$(load_entry "$label") || {
      err "'$label' okunamadı. Yedek iptal edildi."
      return
    }
    local encoded
    encoded=$(printf '%s' "$raw" | base64 -w 0)
    backup_plain+="${encoded}"$'\n'
  done

  # Tüm içeriği tek seferde şifrele
  local encrypted
  encrypted=$(printf '%s' "$backup_plain" |
    openssl enc -aes-256-cbc -pbkdf2 -iter 200000 -pass fd:3 2>/dev/null \
      3<<<"$MASTER_PASS" |
    base64 -w 0)

  if [[ -z "$encrypted" ]]; then
    err "Şifreleme başarısız."
    return
  fi

  printf '%s\n%s\n' "$BACKUP_MAGIC" "$encrypted" >"$out_path"
  chmod 600 "$out_path"

  echo
  success "${#labels[@]} kayıt yedeklendi."
  info "Dosya: ${BOLD}${out_path}${NC}"
  warn "Bu dosyayı güvenli bir ortama (USB, bulut) aktarın."
  warn "Geri yükleme için yedeğin ANA PAROLASINI bilmeniz gerekir."
}

do_restore() {
  echo
  echo -e "  ${BOLD}Yedeği Geri Yükle${NC}"
  hr
  info "Yedek dosyasındaki kayıtlar mevcut depoya aktarılır."
  info "Yedek dosyasının oluşturulduğu ana parola sorulacak."
  echo

  local bak_path
  ask "Yedek dosyası yolu" bak_path
  bak_path="${bak_path/#\~/$HOME}" # ~ genişletme

  if [[ ! -f "$bak_path" ]]; then
    err "Dosya bulunamadı: $bak_path"
    return
  fi

  # Dosya türünü doğrula
  local magic
  magic=$(head -1 "$bak_path")
  if [[ "$magic" != "$BACKUP_MAGIC" ]]; then
    err "Geçersiz ya da bozuk yedek dosyası."
    return
  fi

  local bak_pass
  ask_secret "Yedeğin ana parolası (kaynak bilgisayar)" bak_pass

  # Şifre çöz: base64 → ikili → openssl decrypt
  local encrypted plaintext
  encrypted=$(tail -n +2 "$bak_path")
  plaintext=$(printf '%s' "$encrypted" |
    base64 -d |
    openssl enc -d -aes-256-cbc -pbkdf2 -iter 200000 -pass fd:3 2>/dev/null \
      3<<<"$bak_pass")

  if [[ -z "$plaintext" ]]; then
    err "Şifre çözme başarısız. Parola yanlış ya da dosya bozuk olabilir."
    return
  fi

  # Her satır bir base64 kodlu giriş
  local total=0 imported=0 skipped=0
  while IFS= read -r line; do
    [[ -z "$line" ]] && continue
    ((total++))

    local raw
    raw=$(printf '%s' "$line" | base64 -d 2>/dev/null)
    if [[ -z "$raw" ]]; then
      warn "Giriş #${total} çözülemedi, atlandı."
      ((skipped++))
      continue
    fi

    local label pw url note
    label=$(get_field "$raw" "label")
    pw=$(get_field "$raw" "password")
    url=$(get_field "$raw" "url")
    note=$(get_field "$raw" "note")

    if [[ -z "$label" || -z "$pw" ]]; then
      warn "Giriş #${total}: eksik alan, atlandı."
      ((skipped++))
      continue
    fi

    local file
    file=$(entry_file "$label")
    if [[ -f "$file" ]]; then
      echo -en "  ${YELLOW}!${NC} '${label}' zaten var — üzerine yazılsın mı? [e/H]: "
      read -r _ow
      if [[ "$_ow" =~ ^[EeYy]$ ]]; then
        if save_entry "$label" "$pw" "$url" "$note"; then
          info "'${label}' güncellendi."
          ((imported++))
        else
          ((skipped++))
        fi
      else
        info "'${label}' atlandı."
        ((skipped++))
      fi
    else
      if save_entry "$label" "$pw" "$url" "$note"; then
        ((imported++))
      else
        ((skipped++))
      fi
    fi
  done <<<"$plaintext"

  echo
  success "${imported}/${total} kayıt içe aktarıldı."
  [[ $skipped -gt 0 ]] && warn "${skipped} kayıt atlandı."
}

# ── Ana menü ─────────────────────────────────

main_menu() {
  while true; do
    echo
    hr
    echo -e "  ${BOLD}Ana Menü${NC}"
    hr
    echo -e "  ${DIM}1)${NC} Parola üret ve kaydet"
    echo -e "  ${DIM}2)${NC} Mevcut parolayı kaydet"
    echo -e "  ${DIM}3)${NC} Kayıtları listele"
    echo -e "  ${DIM}4)${NC} Parola göster"
    echo -e "  ${DIM}5)${NC} Kayıt sil"
    echo -e "  ${DIM}6)${NC} Ana parolayı değiştir"
    echo -e "  ${DIM}7)${NC} Yedek al"
    echo -e "  ${DIM}8)${NC} Yedeği geri yükle"
    echo -e "  ${DIM}q)${NC} Çıkış"
    hr

    local choice
    ask "Seçim" choice

    case "$choice" in
    1) do_generate ;;
    2) do_import ;;
    3) do_list ;;
    4) do_show ;;
    5) do_delete ;;
    6) do_change_master ;;
    7) do_backup ;;
    8) do_restore ;;
    q | Q | exit | quit)
      echo
      success "Güle güle."
      echo
      exit 0
      ;;
    *) warn "Geçersiz seçim." ;;
    esac
  done
}

# ── Giriş noktası ─────────────────────────────

main() {
  init_store
  clear
  banner

  if [[ ! -f "$VERIFY_FILE" ]]; then
    setup_master
  else
    echo -e "  ${BOLD}Kilit Aç${NC}"
    hr
    verify_master
    success "Erişim sağlandı."
  fi

  main_menu
}

main
