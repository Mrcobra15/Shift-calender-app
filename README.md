# Shift-calender-appimport streamlit as st
import pandas as pd
from datetime import datetime, date, time, timedelta
from dateutil.relativedelta import relativedelta
import io

st.set_page_config(page_title="Shift Kalender Calculator", page_icon="🗓️", layout="wide")

# ---------- helpers ----------
def parse_hhmm(s: str) -> time:
    """Parse 'HH:MM' to datetime.time, tolerant for 'H:MM'."""
    s = s.strip()
    try:
        h, m = s.split(":")
        return time(int(h), int(m))
    except Exception:
        raise ValueError("Gebruik formaat HH:MM, bv. 06:45")

def hours_between(start: time, end: time) -> float:
    """Return duration in hours between start and end (same day)."""
    dt0 = datetime.combine(date.today(), start)
    dt1 = datetime.combine(date.today(), end)
    # als einde voor start valt, interpreteren we het als 'na middernacht'
    if dt1 <= dt0:
        dt1 += timedelta(days=1)
    return round((dt1 - dt0).total_seconds() / 3600.0, 2)

def month_dates(year: int, month: int):
    d0 = date(year, month, 1)
    d1 = d0 + relativedelta(months=1)
    d = d0
    while d < d1:
        yield d
        d += timedelta(days=1)

def ensure_session():
    if "shiftcodes" not in st.session_state:
        # Voorbeelden uit jouw context
        st.session_state.shiftcodes = {
            "vv": ("06:45", "14:51"),   # vroege shift 7.6u
            "ln": ("15:00", "23:06"),  # late nacht 8.1u
            "Fdrecup": (None, None),   # betaalde feestdag (0u)
        }
    if "data" not in st.session_state:
        st.session_state.data = pd.DataFrame()

ensure_session()

# ---------- sidebar: maand & shiftcodes ----------
with st.sidebar:
    st.header("📅 Maand kiezen")
    today = date.today()
    colA, colB = st.columns(2)
    year = colA.number_input("Jaar", min_value=2000, max_value=2100, value=today.year, step=1)
    month = colB.number_input("Maand", min_value=1, max_value=12, value=today.month, step=1)

    st.divider()
    st.header("⚙️ Shiftcodes beheren")

    with st.form("shift_form", clear_on_submit=False):
        code = st.text_input("Code (bv. vv, ln, Fdrecup)").strip()
        start = st.text_input("Start (HH:MM, leeg laten bij 0u)", value="")
        einde = st.text_input("Einde (HH:MM, leeg laten bij 0u)", value="")
        submitted = st.form_submit_button("Toevoegen/Updaten")
        if submitted:
            if not code:
                st.error("Geef een code op.")
            else:
                # None als leeg => 0u-shift
                s = start.strip() or None
                e = einde.strip() or None
                if (s and not e) or (e and not s):
                    st.error("Start én einde invullen, of beide leeg voor 0u.")
                else:
                    if s and e:
                        # validatie
                        try:
                            parse_hhmm(s); parse_hhmm(e)
                        except Exception as ex:
                            st.error(str(ex))
                        else:
                            st.session_state.shiftcodes[code] = (s, e)
                            st.success(f"Shiftcode '{code}' opgeslagen.")
                    else:
                        st.session_state.shiftcodes[code] = (None, None)
                        st.success(f"Shiftcode '{code}' (0u) opgeslagen.")

    st.write("📌 **Beschikbare shiftcodes:**")
    st.json(st.session_state.shiftcodes)

# ---------- hoofd: data-invoer ----------
st.title("🗓️ Shift Kalender Calculator")

# initialiseer maandtabel als die nog leeg is of maand/jr veranderd
key_for_month = f"{year}-{month:02d}"
if st.session_state.data.empty or st.session_state.data.get("MaandKey", pd.Series(dtype=str)).ne(key_for_month).all():
    rows = []
    for d in month_dates(year, month):
        rows.append({
            "Datum": pd.to_datetime(d),
            "Dag": d.strftime("%a"),
            "Code": "",
            "ExtraUren": 0.0,
            "Notities": "",
        })
    df = pd.DataFrame(rows)
    df["MaandKey"] = key_for_month
    st.session_state.data = df
else:
    df = st.session_state.data.copy()

st.markdown("### Maandoverzicht (bewerkbaar)")
st.caption("Vul per dag een **Code** (uit de sidebar) en optioneel **ExtraUren** (decimaal, bv. 1.5). Laat leeg voor vrije dag.")

edited = st.data_editor(
    df[["Datum", "Dag", "Code", "ExtraUren", "Notities"]],
    num_rows="fixed",
    hide_index=True,
    use_container_width=True,
    key="editor"
)

# terugschrijven
df[["Datum", "Dag", "Code", "ExtraUren", "Notities"]] = edited

# ---------- berekenen ----------
def calc_hours_for_row(code: str) -> float:
    code = (code or "").strip()
    if not code:
        return 0.0
    sc = st.session_state.shiftcodes.get(code)
    if not sc:
        # onbekende code => 0u, markeer later
        return 0.0
    s, e = sc
    if s is None and e is None:
        return 0.0
    return hours_between(parse_hhmm(s), parse_hhmm(e))

df["ShiftUren"] = df["Code"].apply(calc_hours_for_row)
df["TotaalUren"] = (df["ShiftUren"].fillna(0) + df["ExtraUren"].fillna(0)).round(2)
df["Week"] = df["Datum"].dt.isocalendar().week.astype(int)

# waarschuwingen
unknown_codes = sorted(set([c for c in df["Code"] if c and c not in st.session_state.shiftcodes]))
if unknown_codes:
    st.warning(f"Onbekende codes in de tabel: {', '.join(unknown_codes)}. Voeg ze eerst toe in de sidebar.")

# ---------- resultaten ----------
st.subheader("📊 Resultaten")
col1, col2, col3 = st.columns(3)
col1.metric("Maand totaal (uren)", f"{df['TotaalUren'].sum():.2f}")
col2.metric("Gem. per gewerkte dag", f"{df.loc[df['TotaalUren']>0, 'TotaalUren'].mean():.2f}" if (df["TotaalUren"]>0).any() else "0.00")
col3.metric("Aantal gewerkte dagen", int((df["TotaalUren"]>0).sum()))

st.markdown("#### Per week")
per_week = df.groupby("Week", as_index=False)["TotaalUren"].sum().sort_values("Week")
st.dataframe(per_week, use_container_width=True)

st.markdown("#### Detail (met berekende uren)")
detail = df.copy()
detail = detail[["Datum", "Dag", "Code", "ShiftUren", "ExtraUren", "TotaalUren", "Week", "Notities"]].sort_values("Datum")
st.dataframe(detail, use_container_width=True)

# ---------- export ----------
csv_bytes = detail.to_csv(index=False).encode("utf-8")
st.download_button(
    "⬇️ Download CSV",
    data=csv_bytes,
    file_name=f"shift_overzicht_{year}_{month:02d}.csv",
    mime="text/csv"
)

st.caption("Tip: voeg je standaardcodes toe in de sidebar. Onbekende codes tellen 0 uur tot je ze definieert.")