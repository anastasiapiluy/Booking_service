# Booking_service

## Пароль для всех существующих организаций: 123

### Программа
#### Classes.py
```python
import hashlib
import json
import secrets
import datetime

class Organization:
    def __init__(self, id, name, hash="", salt=""):
        self.id = id
        self.name = name
        self.hash = hash
        self.salt = salt

    def create_password(self, password):
        self.salt = secrets.token_hex(16)
        self.hash = hashlib.sha256((password+self.salt).encode("utf-8")).hexdigest()

    def check_password(self, password):
        new_hash = hashlib.sha256((password+self.salt).encode("utf-8")).hexdigest()
        return new_hash == self.hash

    def to_dict(self):
        return {"id":self.id, "name":self.name, "password_hash":self.hash, "salt":self.salt}

    @classmethod
    def from_dict(cls, data):
        return cls(id=data["id"], name=data["name"], hash=data["password_hash"], salt=data["salt"])

class Hall:
    def __init__(self,name, number, open_time = 540, close_time = 1260, cell_amount = 48, cell_time = 15):
        self.name = name
        self.number = number
        self.open_time = open_time
        self.close_time = close_time
        self.cell_amount = cell_amount
        self.cell_time = cell_time

    def to_time(self, number):
        total = self.open_time + self.cell_time*number
        hours = total // 60
        minutes = total % 60
        return f"{hours:02}:{minutes:02}"

    def to_dict(self):
        return {"number": self.number}

class Event:
    def __init__(self, id, hall_number, date, name, organization_id, start_cell, end_cell, open_time = 540, close_time = 1260, cell_time=15):
        self.id = id
        self.hall_number = hall_number
        if isinstance(date, str):
            self.date = datetime.date.fromisoformat(date)
        else:
            self.date = date
        self.name = name
        self.organization_id = organization_id
        self.start_cell = start_cell
        self.end_cell = end_cell
        self.open_time = open_time
        self.close_time = close_time
        self.cell_time = cell_time

    def overlapped(self, new_start, new_end):
        return not (new_start >= self.end_cell or new_end <= self.start_cell)

    def is_now (self, time):
        start = self.open_time+self.start_cell*self.cell_time
        end = self.open_time+self.end_cell*self.cell_time
        return start <= time < end

    def is_soon(self, time):
        start = self.open_time+self.start_cell*self.cell_time
        return time <= start <= time+60

    def to_dict(self):
        return {"id":self.id, "organization_id":self.organization_id, "hall_number":self.hall_number, "date":self.date.isoformat(), "name":self.name, "start_cell":self.start_cell, "end_cell":self.end_cell}

    def get_range(self):
        start = self.open_time+self.start_cell*self.cell_time
        end = self.open_time+self.end_cell*self.cell_time
        return f"{start//60:02}:{start%60:02}-{end//60:02}:{end%60:02}"

    @classmethod
    def from_dict(cls, data):
        return cls(id=data["id"], hall_number=data["hall_number"], name=data["name"], date=data["date"], organization_id=data["organization_id"], start_cell=data["start_cell"], end_cell=data["end_cell"])

class Store:
    def __init__(self):
        self.organizations = {}
        self.halls = {}
        self.events = {}
        self.event_id = 1
        self.halls = {1:Hall("Hall 1", 1), 2:Hall("Hall 2", 2), 3:Hall("Hall 3", 3), 4:Hall("Hall 4", 4), 5:Hall("Hall 5", 5)}
        self.load()

    def load(self):
        try:
            with open("data.json","r", encoding="utf-8") as f:
                data = json.load(f)
                for organization in data["organizations"]:
                    org = Organization.from_dict(organization)
                    self.organizations[org.id] = org
                for event in data["events"]:
                    ev = Event.from_dict(event)
                    self.events[ev.id] = ev
                self.event_id = data.get("event_id",1)
        except Exception as e:
            print(e)

    def save(self):
        data = {"organizations":[organization.to_dict() for organization in self.organizations.values()], "events":[event.to_dict() for event in self.events.values()], "event_id":self.event_id}
        with open("data.json","w", encoding="utf-8") as f:
            json.dump(data, f, indent=4)

    def add_event(self, event):
        event.id = self.event_id
        self.event_id += 1
        self.events[event.id] = event
        self.save()
        return event

    def delete_event(self, event_id):
        if event_id in self.events:
            del self.events[event_id]
            self.save()

    def get_day_events(self, hall_number, date):
        return [event for event in self.events.values() if event.hall_number == hall_number and event.date == date]


class Services:
    def __init__(self, store, user):
        self.store = store
        self.user = user

    def login(self, name, password):
        for organization in self.store.organizations.values():
            if organization.name == name:
                if organization.check_password(password):
                    self.user = organization
                    return True
                else:
                    raise Exception("Invalid password")
        raise Exception("Organization not found")

    def logout(self):
        self.user = None

    def is_admin(self):
        return self.user is not None

    def create_booking(self, hall_number, date, start_cell, end_cell, name, guest_id=None):
        if self.is_admin():
            organization_id = self.user.id
        else:
            organization_id = guest_id
        day_events = self.store.get_day_events(hall_number, date)
        for event in day_events:
            if event.overlapped(start_cell, end_cell):
                raise Exception(f"This time is busy: {event.name}")
        new_event = Event(None, hall_number, date, name, organization_id, start_cell, end_cell)
        return self.store.add_event(new_event)

    def change_booking(self, event_id, start, end, name):
        if not self.is_admin():
            raise Exception("Authorization is required")
        event = self.store.events.get(event_id)
        if not event:
            raise Exception("Event not found")
        if event.organization_id != self.user.id:
            raise Exception("You can change only your organization's booking")
        day_events = self.store.get_day_events(event.hall_number, event.date)
        for ev in day_events:
            if ev.id != event_id and ev.overlapped(start, end):
                raise Exception(f"This time is busy: {ev.name}")
        event.start_cell = start
        event.end_cell = end
        event.name = name
        self.store.save()

    def delete_event(self, event_id):
        if not self.is_admin():
            raise Exception("Authorization is required")
        event = self.store.events.get(event_id)
        if not event:
            raise Exception("Event not found")
        if event.organization_id != self.user.id:
            raise Exception("You can change only your organization's booking")
        self.store.delete_event(event_id)

```
#### Interface.py
```python
import calendar
from tkinter import *
from tkinter import messagebox
from tkinter.ttk import Notebook
from tkcalendar import Calendar
from datetime import date, datetime
from Classes import Store, Event, Services, Organization, Hall

store = Store()
user = None
services = Services(store, user)
chosen_date = date.today()
start_cell = None
end_cell = None
chosen_hall = 1
cells = {1:[],2:[],3:[],4:[],5:[]}
time_ent = None
editing_id = None

def choose_cell(cell):
    global start_cell, end_cell, time_ent
    i = cell.widget.index
    chosen = cell.widget
    if chosen["bg"]=="red":
        events = store.get_day_events(chosen_hall, chosen_date)
        for event in events:
            if event.start_cell <= i < event.end_cell:
                organization_name = store.organizations[event.organization_id].name
                messagebox.showinfo("Information", f"This time is busy by {organization_name}")
                break
        return
    for j in range(48):
        if cells[chosen_hall][j]["bg"]=="blue":
            cells[chosen_hall][j].config(background="green")
    if start_cell is None:
        start_cell = i
        end_cell = i+1
    elif end_cell==start_cell+1:
        if i>=start_cell:
            end_cell = i+1
        else:
            start_cell = i
            end_cell = i+1
    else:
        start_cell = i
        end_cell = i+1
    update_timeline(chosen_date, chosen_hall)

    if start_cell is not None:
        if end_cell is not None:
            for j in range(start_cell, end_cell):
                if cells[chosen_hall][j]["bg"] != "red":
                    cells[chosen_hall][j].config(background="blue")
        else:
            if cells[chosen_hall][start_cell]["bg"] != "red":
                cells[chosen_hall][start_cell].config(background="blue")
    hall=store.halls[chosen_hall]
    time_entry.config(state="normal")
    time_entry.delete(0, END)
    time_entry.insert(0, f"{hall.to_time(start_cell)} - {hall.to_time(end_cell)}")
    time_entry.config(state="readonly")
    if time_ent is not None:
        try:
            time_ent.config(state="normal")
            time_ent.delete(0, END)
            time_ent.insert(0,f"{hall.to_time(start_cell)} - {hall.to_time(end_cell)}")
            time_ent.config(state="readonly")
        except Exception as e:
            messagebox.showerror("Error", f"{e}")

def update_timeline(chosen_date, chosen_hall):
    for i in range(48):
        cells[chosen_hall][i].config(background="green")
    ev = store.get_day_events(chosen_hall, chosen_date)
    for event in ev:
        color="red"
        if editing_id is not None and event.id==editing_id:
            color="yellow"
        for i in range(event.start_cell, event.end_cell):
            cells[chosen_hall][i].config(background=color)

def update_events():
    global current_id
    event_listbox.delete(0, END)
    current_id = []
    events = store.get_day_events(chosen_hall, chosen_date)
    events.sort(key=lambda x: x.start_cell)
    for event in events:
        organization_name = store.organizations[event.organization_id].name
        time = event.get_range()
        heading = f"{time}, {organization_name}, {event.name}"
        event_listbox.insert(END, heading)
        current_id.append(event.id)

def delete_event():
    chosen = event_listbox.curselection()
    if not chosen:
        messagebox.showerror("Error", "No events selected")
        return
    if not services.is_admin():
        messagebox.showerror("Error", "No user selected")
        return
    id = current_id[chosen[0]]
    try:
        services.delete_event(id)
        messagebox.showinfo("Success", "Event deleted")
        update_timeline(chosen_date, chosen_hall)
        update_events()
    except Exception as e:
        messagebox.showerror("Error", f"Event not deleted: {e}")

def change_event():
    global start_cell, end_cell, time_ent, editing_id
    chosen = event_listbox.curselection()
    if not chosen:
        messagebox.showerror("Error", "No events selected")
        return
    id = current_id[chosen[0]]
    event=store.events.get(id)
    if not services.is_admin():
        messagebox.showerror("Error", "Authorization is required!")
        return
    if event.organization_id!=services.user.id:
        messagebox.showerror("Error", "You can change only your organization's events")
        return
    editing_id = id
    start_cell = None
    end_cell = None
    update_timeline(chosen_date, chosen_hall)
    try:
        changing_window = Toplevel(root)
        changing_window.title("Change event")
        changing_window.geometry("300x200")

        Label(changing_window, text="New name: ").grid(row=0, column=0)
        name_ent = Entry(changing_window)
        name_ent.insert(0,event.name)
        name_ent.grid(row=0, column=1)
        Label(changing_window, text="New time: ").grid(row=1, column=0)
        loc_time_entry = Entry(changing_window, state="readonly")
        if start_cell is not None and end_cell is not None:
            new_time = f"{store.halls[chosen_hall].to_time(start_cell)} - {store.halls[chosen_hall].to_time(end_cell)}"
        else:
            new_time = event.get_range()

        loc_time_entry.config(state="normal")
        loc_time_entry.insert(0, new_time)
        loc_time_entry.config(state="readonly")
        loc_time_entry.grid(row=1, column=1)
        time_ent = loc_time_entry

        def close_window():
            global time_ent, start_cell, end_cell, editing_id
            time_ent = None
            start_cell = None
            end_cell = None
            editing_id = None
            update_timeline(chosen_date, chosen_hall)
            clean_entry(time_entry)
            for j in range(48):
                if cells[chosen_hall][j]["bg"]=="blue":
                    cells[chosen_hall][j].config(background="green")
            changing_window.destroy()

        def save_event():
            name = name_ent.get().strip()
            if name=="":
                messagebox.showerror("Error", "Please enter a name")
                return
            if start_cell is not None and end_cell is not None:
                    end = end_cell
                    start = start_cell
            else:
                start = event.start_cell
                end = event.end_cell
            try:
                services.change_booking(id, start, end, name)
                messagebox.showinfo("Success", "Event changed")
                close_window()
                update_timeline(chosen_date, chosen_hall)
                update_events()

            except Exception as e:
                messagebox.showerror("Error", f"Event not changed, {e}")

        Button(changing_window, text="Change event", command=save_event).grid(row=2, column=0)
        changing_window.protocol("WM_DELETE_WINDOW", close_window)
    except Exception as e:
        messagebox.showerror("Error", str(e))

def clean_entry(entry):
    entry.config(state="normal")
    entry.delete(0, END)
    entry.config(state="readonly")

def change_date(event):
    global chosen_date, start_cell, end_cell
    chosen_date = calendar.selection_get()
    try:
        date_entry.config(state="normal")
        date_entry.delete(0, END)
        date_entry.insert(0, chosen_date)
        date_entry.config(state="readonly")
    except:
        pass
    start_cell = None
    end_cell = None
    clean_entry(time_entry)
    update_timeline(chosen_date, chosen_hall)
    update_events()

def change_hall(event=None):
    global chosen_hall, start_cell, end_cell
    chosen_hall = notebook.index(notebook.select())+1
    start_cell = None
    end_cell = None
    clean_entry(time_entry)
    update_timeline(chosen_date, chosen_hall)
    update_events()

def add_organization():
    name = new_org_entry.get()
    password = pass_entry.get()
    if not name:
        messagebox.showerror("Error", "Please enter an organization name")
        return
    if not password:
        messagebox.showerror("Error", "Please enter a password")
        return
    for org in store.organizations.values():
        if org.name==name:
            messagebox.showerror("Error", "Organization already exists")
            return
    id = max(store.organizations.keys(), default=0)+1
    new_org = Organization(id, name)
    new_org.create_password(password)
    store.organizations[id] = new_org
    store.save()
    org_entry.insert(END, new_org.name)
    new_org_entry.delete(0, END)
    pass_entry.delete(0, END)
    messagebox.showinfo("Success", "Organization created")

def book():
    global start_cell, end_cell, chosen_hall, chosen_date
    name = name_entry.get()
    if name=="":
        messagebox.showerror("Error", "Please enter a name")
        return
    if start_cell is None:
        messagebox.showerror("Error", "Please choose time gap")
        return
    organization_id = None
    if not services.is_admin():
        try:
            organization_name = org_entry.get(org_entry.curselection()[0])
        except Exception as e:
            messagebox.showerror("Error", str(e))
            return
        for org in store.organizations.values():
            if org.name==organization_name:
                organization_id = org.id
                break
        if not organization_id:
            messagebox.showerror("Error", "Please choose an organization!")
            return
    try:
        services.create_booking(chosen_hall, chosen_date,start_cell, end_cell,name,organization_id)
        messagebox.showinfo("Success", "Booking created")
        start_cell = None
        end_cell = None
        name_entry.delete(0, END)
        clean_entry(time_entry)
        update_timeline(chosen_date, chosen_hall)
        update_events()
    except Exception as e:
        messagebox.showerror("Error", f"Booking not created, {e}")

def interface():
    global user
    login_window = Toplevel(root)
    login_window.title("Log In as an organization")
    login_window.geometry("280x200")
    login_window.resizable(width=False, height=False)
    login_window.grab_set()

    Label(login_window, text="Username: ").grid(row=0, column=0, padx=5, pady=2)
    username_entry = Entry(login_window, width=25)
    username_entry.grid(row=0, column=1, padx=5, pady=5)
    Label(login_window, text="Password: ").grid(row=1, column=0, padx=5, pady=2)
    password_entry = Entry(login_window, width=25, show="*")
    password_entry.grid(row=1, column=1, padx=5, pady=2)

    def login():
        try:
            services.login(username_entry.get(), password_entry.get())
            org_entry.config(state="disabled")
            state.config(text=f"Logged in as {username_entry.get()}")
            login_window.destroy()
        except Exception as e:
            messagebox.showerror("Error", f"Login not successful, {e}")

    def logout():
        try:
            services.logout()
            org_entry.config(state="normal")
            state.config(text="Status: Guest", background="red")
            login_window.destroy()
        except Exception as e:
            messagebox.showerror("Error", f"Logout not successful, {e}")

    Button(login_window, text="Confirm", command=login).grid(row=2, column=0, padx=5, pady=20)
    Button(login_window, text="Log Out", command=logout).grid(row=2, column=1, padx=5, pady=20)

def dashboard():
    now = datetime.now()
    current_time = now.hour*60+now.minute
    today = date.today()
    dash_listbox.delete(0, END)
    flag=False
    for event in store.events.values():
        if event.date==today:
            if event.is_now(current_time):
                dash_listbox.insert(END, f"NOW: Hall {event.hall_number} - {event.name}")
                flag = True
            elif event.is_soon(current_time):
                dash_listbox.insert(END, f"SOON: Hall {event.hall_number} - {event.name}")
                flag = True
    if not flag:
        dash_listbox.insert(END, "There is no events soon")
    root.after(60000, dashboard)

root = Tk()
root.geometry("1200x700")
root.title("Hall Booking")

down_frame = Frame(root, bg="grey", height=180)
down_frame.pack(side="bottom", fill="x")
down_frame.pack_propagate(False)
Label(down_frame, text="DASHBOARD", background="yellow").pack(side="top", padx=5, pady=5)
dash_listbox = Listbox(down_frame, height = 10)
dash_listbox.pack(side="top", fill="x", padx=5, pady=5)

left_frame = Frame(root)
left_frame.pack(side="left", fill="both", expand=True, padx=5, pady=5)
top_frame = Frame(left_frame)
top_frame.pack(side="top", fill="x", expand=True)
book_frame = Frame(top_frame)
book_frame.pack(side="left", fill="both", expand=True)

state=Label(book_frame, text="Status: Guest", bg="red")
state.grid(row=0, column=0, columnspan=2,padx=5, pady=5)
Label(book_frame, text="If your organization is not in the list, \n you can add it here:", foreground="blue").grid(row=0, column=4, columnspan=2, padx=5, pady=5)
Label(book_frame, text="Event name: ").grid(row=1, column=0, padx=5, pady=2)
name_entry = Entry(book_frame, width=25)
name_entry.grid(row=1, column=1, padx=5, pady=5)

Label(book_frame, text="Organization name: ").grid(row=2, column=0, padx=5)
org_entry = Listbox(book_frame, width=25, height=5)
org_entry.grid(row=2, column=1, padx=5, pady=5)
for org in store.organizations.values():
    org_entry.insert(END, org.name)
Label(book_frame, text="New organization name: ").grid(row=1, column=4, padx=5)
new_org_entry = Entry(book_frame, width=20)
new_org_entry.grid(row=1, column=5, padx=5, pady=5)
Label(book_frame, text="New password: ").grid(row=2, column=4, padx=5)
pass_entry = Entry(book_frame, width=20, show="*")
pass_entry.grid(row=2, column=5, padx=5)
Button(book_frame, text="Add new organization", command=add_organization).grid(row=3, column=5, padx=5, pady=5)

date_label=Label(book_frame, text="Date: ")
date_label.grid(row=1, column=2, padx=5, pady=5)
date_entry=Entry(book_frame, state="readonly")
date_entry.grid(row=1, column=3, padx=5, pady=5)
time_label=Label(book_frame, text="Time: ")
time_label.grid(row=2, column=2, padx=5, pady=5)
time_entry = Entry(book_frame, width=20, state="readonly")
time_entry.grid(row=2, column=3, padx=5, pady=5)

Button(book_frame, text="Book", bg="green",command=book).grid(row=3, column=0,columnspan=4, padx=5)

calendar_frame = Frame(top_frame)
calendar_frame.pack(side="right", padx=10)
calendar = Calendar(calendar_frame, selectmode="day")
calendar.bind("<<CalendarSelected>>", change_date)
calendar.pack(padx=5, pady=5)

login_frame = Frame(book_frame)
login_frame.grid(row=4, column=0,columnspan=4, padx=5, pady=5)
Button(login_frame, text="Login/Logout", command=interface).pack(side="left", padx=5, pady=5)

right_frame = Frame(root, bg="grey",width=300, height=500)
right_frame.pack(side = "right", padx=5,pady=5)
right_frame.pack_propagate(False)
Label(right_frame, text="Events").pack(side="top", pady=5)
event_listbox = Listbox(right_frame, height = 12)
event_listbox.pack(side="top", fill="both", expand=True,padx=10,pady=5)
current_id = []
Button(right_frame, text="Change chosen event", command=change_event).pack(side="top", fill="x", padx=5, pady=5)
Button(right_frame, text="Delete chosen event", command=delete_event).pack(side="top", fill="x", padx=5, pady=5)

notebook = Notebook(left_frame)
notebook.pack(fill="both", expand=True,pady=10)
notebook.bind("<<NotebookTabChanged>>", change_hall)
parts = []

for hall in range(1,6):
    window = Frame(notebook)
    notebook.add(window, text=f"Hall {hall}")
    time_frame = Frame(window)
    time_frame.pack(expand=True, fill="both", padx=5, pady=5)
    for row in range(4):
        for column in range(24):
            i = row//2 * 24 + column
            if row%2==0:
                Label(time_frame, text=str(store.halls[hall].to_time(i))).grid(row=row, column=column)
            else:
                part = Frame(time_frame, width=40, height = 40, bg="green", pady=2)
                part.grid(row=row, column=column)
                part.index=i
                part.bind("<Button-1>", choose_cell)
                cells[hall].append(part)
    for column in range(24):
        time_frame.grid_columnconfigure(column, weight=1)
    for row in range(2):
        time_frame.grid_rowconfigure(row, weight=0)

update_timeline(chosen_date, chosen_hall)
update_events()
dashboard()

root.mainloop()
```
