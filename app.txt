mport hmac
import os

import mysql.connector
import streamlit as st
from mysql.connector import Error as MySQLError


st.set_page_config(
    page_title="Woldia University Admin Dashboard",
    page_icon="🎓",
    layout="wide",
)


def require_dashboard_password() -> None:
    expected_password = os.environ.get("DASHBOARD_PASSWORD", "")
    if not expected_password:
        st.error("DASHBOARD_PASSWORD is not configured. Set it in the deployment environment.")
        st.stop()

    if st.session_state.get("authenticated"):
        return

    st.title("Woldia University Admin Dashboard")
    with st.form("dashboard_login"):
        password = st.text_input("Dashboard password", type="password")
        submitted = st.form_submit_button("Sign in")

    if submitted:
        if hmac.compare_digest(password, expected_password):
            st.session_state["authenticated"] = True
            st.rerun()
        else:
            st.error("Incorrect password.")
    st.stop()


def database_config() -> dict[str, object]:
    required = ("DB_HOST", "DB_NAME", "DB_USER", "DB_PASSWORD")
    missing = [name for name in required if not os.environ.get(name)]
    if missing:
        raise ValueError(f"Missing database settings: {', '.join(missing)}")

    try:
        port = int(os.environ.get("DB_PORT", "3306"))
    except ValueError as error:
        raise ValueError("DB_PORT must be a valid port number.") from error

    if not 1 <= port <= 65535:
        raise ValueError("DB_PORT must be between 1 and 65535.")

    return {
        "host": os.environ["DB_HOST"],
        "port": port,
        "database": os.environ["DB_NAME"],
        "user": os.environ["DB_USER"],
        "password": os.environ["DB_PASSWORD"],
        "connection_timeout": 10,
        "charset": "utf8mb4",
    }


def load_metrics(config: dict[str, object]) -> dict[str, int]:
    query = """
        SELECT
            (SELECT COUNT(*) FROM users) AS total_users,
            (SELECT COUNT(*) FROM users WHERE role = 'student') AS total_students,
            (SELECT COUNT(*) FROM users WHERE role = 'staff') AS total_staff,
            (SELECT COUNT(*) FROM complaints) AS total_complaints,
            (SELECT COUNT(*) FROM announcements) AS total_announcements,
            (SELECT COUNT(*) FROM student_clearance) AS total_clearances
    """
    connection = mysql.connector.connect(**config)
    cursor = connection.cursor(dictionary=True)
    try:
        cursor.execute(query)
        row = cursor.fetchone()
        if row is None:
            raise RuntimeError("The dashboard metrics query returned no result.")
        return {name: int(value) for name, value in row.items()}
    finally:
        cursor.close()
        connection.close()


require_dashboard_password()

st.title("Woldia University Admin Dashboard")
st.caption("Live summary from the campus portal database")

try:
    metrics = load_metrics(database_config())
except (MySQLError, ValueError, RuntimeError) as error:
    st.error(f"Unable to load dashboard data: {error}")
    st.stop()

metric_groups = (
    (
        ("Total users", "total_users"),
        ("Students", "total_students"),
        ("Staff", "total_staff"),
    ),
    (
        ("Complaints", "total_complaints"),
        ("Announcements", "total_announcements"),
        ("Clearances", "total_clearances"),
    ),
)

for group in metric_groups:
    columns = st.columns(len(group))
    for column, (label, key) in zip(columns, group):
        column.metric(label, metrics[key])

st.subheader("People")
st.bar_chart(
    {
        "People": {
            "Students": metrics["total_students"],
            "Staff": metrics["total_staff"],
        }
    },
    x_label="Role",
    y_label="Users",
)
