import ast
import os
import shutil
import sys
from collections import defaultdict
from copy import deepcopy
import tkinter as tk
from functools import reduce
from tkinter import ttk
from configparser import ConfigParser
import re
from flatten_any_dict_iterable_or_whatsoever import fla_tu, set_in_original_iter
from flatten_everything import flatten_everything
from tolerant_isinstance import isinstance_tolerant
nested_dict = lambda: defaultdict(nested_dict)
allvars = sys.modules[__name__]
reg = re.compile(r"^[-.,\de+]+", flags=re.I)

regint = re.compile(r"^[-\d+]+", flags=re.I)
from deepcopyall import deepcopy


def load_config_file_vars(cfgfile, onezeroasboolean):
    pars2 = ConfigParser()
    pars2.read(cfgfile)

    (
        cfgdictcopy,
        cfgdictcopyaslist,
        cfgdictcopysorted,
        maxlenlist,
        labellength,
        sublablelength,
        maxlenabs,
        maxlenabsvals,
    ) = copy_dict_and_convert_values(pars2, onezeroasboolean=onezeroasboolean)

    for key, item in cfgdictcopyaslist:
        try:
            setattr(sys.modules["__main__"], item[-1], key)
        except Exception:
            pass
            # print(f"ERROR! variable: {item[-1]}, value: {key}")


def convert_to_normal_dict(di):
    if isinstance_tolerant(di, defaultdict):
        di = {k: convert_to_normal_dict(v) for k, v in di.items()}
    return di


def groupBy(key, seq, continue_on_exceptions=True, withindex=True, withvalue=True):
    indexcounter = -1

    def execute_f(k, v):
        nonlocal indexcounter
        indexcounter += 1
        try:
            return k(v)
        except Exception as fa:
            if continue_on_exceptions:
                return "EXCEPTION: " + str(fa)
            else:
                raise fa

    # based on https://stackoverflow.com/a/60282640/15096247
    if withvalue:
        return convert_to_normal_dict(
            reduce(
                lambda grp, val: grp[execute_f(key, val)].append(
                    val if not withindex else (indexcounter, val)
                )
                or grp,
                seq,
                defaultdict(list),
            )
        )
    return convert_to_normal_dict(
        reduce(
            lambda grp, val: grp[execute_f(key, val)].append(indexcounter) or grp,
            seq,
            defaultdict(list),
        )
    )


def groupby_first_item(
    seq, continue_on_exceptions=True, withindex=False, withvalue=True
):
    return groupBy(
        key=lambda x: x[0],
        seq=seq,
        continue_on_exceptions=continue_on_exceptions,
        withindex=withindex,
        withvalue=withvalue,
    )


def copy_dict_and_convert_values(pars, onezeroasboolean=False):
    copieddict = deepcopy(pars.__dict__["_sections"])
    flattli = fla_tu(pars.__dict__["_sections"])
    for value, keys in flattli:
        if not re.search(r"^(?:[01])$", str(value)):
            try:
                valuewithdtype = pars.getboolean(*keys)
            except Exception:
                try:
                    valuewithdtype = ast.literal_eval(pars.get(*keys))
                except Exception:
                    valuewithdtype = pars.get(*keys)
        else:
            if onezeroasboolean:
                valuewithdtype = pars.getboolean(*keys)
            else:
                valuewithdtype = ast.literal_eval(pars.get(*keys))

        set_in_original_iter(iterable=copieddict, keys=keys, value=valuewithdtype)

    g = list(fla_tu(copieddict))
    gr = groupby_first_item([(x[1][0], (x[0], x[1])) for x in g])
    maxlenlist = [int(max([len(x[1]) for x in gr.items()]) * 2) + 5] * len(gr)
    labellength = max([len(x) for x in gr]) + 1
    sublablelength = max([len(x[1][-1]) for x in g])
    maxlenabs = max([len(xx) for xx in list(flatten_everything([x[1] for x in g]))])
    maxlenabsvals = max(
        [len(str(xx)) for xx in list(flatten_everything([x[0] for x in g]))]
    )

    return (
        copieddict,
        g,
        gr,
        maxlenlist,
        labellength,
        sublablelength,
        maxlenabs,
        maxlenabsvals,
    )


def validate_ast_float(x):
    try:
        if reg.search(str(x)):
            return ast.literal_eval(str(x))
    except Exception as fe:
        return x
    return x


def validate_ast_int(x):
    try:
        if regint.search(str(x)):
            return ast.literal_eval(str(x))
    except Exception as fe:
        return x
    return x


def on_validate_float(P):
    va = validate_ast_float(P)
    if isinstance_tolerant(va, int) or isinstance_tolerant(va, float):
        return True
    return False


def on_validate_int(P):
    va = validate_ast_int(P)
    if isinstance_tolerant(va, int):
        return True
    return False


def on_validate_bool(P):
    if str(P) in ["0", "1"]:
        # print(str(P))
        return True
    return False


class Cfedit:
    def __init__(
        self,
        cfgfile,
        title,
        icon=None,
        res="1024x768",
        addbuttons=(),
        mainlabelfont=("Helvetica", 15, "underline bold italic"),
        sublabelfont=("Helvetica", 14),
        varfont=("Helvetica", 10),
        buttonfont=("Helvetica", 12, "bold italic"),
    ):
        cfgfile = os.path.normpath(cfgfile)
        backupfile = re.sub(r"\W+", "", cfgfile)
        self.backupconfig = os.path.normpath(
            os.path.join(
                os.path.dirname(__file__), f"{backupfile}__cfgbackupCONFIG.cof"
            )
        )
        if not os.path.exists(self.backupconfig):
            shutil.copy2(cfgfile, self.backupconfig)
        self.title = title
        self.check_all_vars = []
        self.pars = ConfigParser()
        self.pars.read(cfgfile)
        self.cfgfile = cfgfile
        self.onezeroasboolean = False
        self.addbuttons = addbuttons
        (
            self.cfgdictcopy,
            self.cfgdictcopyaslist,
            self.cfgdictcopysorted,
            self.maxlenlist,
            self.labellength,
            self.sublablelength,
            self.maxlenabs,
            self.maxlenabsvals,
        ) = copy_dict_and_convert_values(
            self.pars, onezeroasboolean=self.onezeroasboolean
        )
        self.res = res
        self.root = tk.Tk()
        self.root.geometry(self.res)
        self.icon = icon
        if icon:
            self.root.iconphoto(False, tk.PhotoImage(file=os.path.normpath(icon)))
        self.root.title(self.title)
        # CANVAS / FRAME XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
        self.mainframe = tk.Frame(self.root)
        self.canvas_in_mainframe = tk.Canvas(self.mainframe)
        self.scrollbar_right_side = ttk.Scrollbar(
            self.mainframe,
            orient=tk.VERTICAL,
            command=self.canvas_in_mainframe.yview,
        )
        self.subframe_in_canvas = tk.Frame(self.canvas_in_mainframe)
        self.alladdedobjects = nested_dict()
        self.mainlabelfont = mainlabelfont
        self.sublabelfont = sublabelfont
        self.varfont = varfont
        self.buttonfont = buttonfont
        # XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

    def load_variables_from_cfgfile_to_global(self, onezeroasboolean=True):
        load_config_file_vars(cfgfile=self.cfgfile, onezeroasboolean=onezeroasboolean)

    def exit_config_start_app(self):
        self.root.destroy()
        try:
            self.mainframe.destroy()
        except Exception:
            pass
        try:

            self.canvas_in_mainframe.destroy()
        except Exception:
            pass
        try:
            self.subframe_in_canvas.destroy()
        except Exception:
            pass
        try:
            self.scrollbar_right_side.destroy()
        except Exception:
            pass

    def exit_app(self):
        self.exit_config_start_app()
        os._exit(0)

    def refresh(self):
        self.exit_config_start_app()

        self.__init__(
            cfgfile=self.cfgfile,
            title=self.title,
            icon=self.icon,
            res=self.res,
            addbuttons=self.addbuttons,
            mainlabelfont=self.mainlabelfont,
            sublabelfont=self.sublabelfont,
            varfont=self.varfont,
            buttonfont=self.buttonfont,
        )
        self.mainloop()

    def _restore_cfg(self):
        if os.path.exists(self.cfgfile):
            os.remove(self.cfgfile)
        shutil.copy2(self.backupconfig, self.cfgfile)
        self.refresh()

    def _update_cfg(self):
        for li in self.check_all_vars:
            varab = li[-1]()
            if self.onezeroasboolean:
                if isinstance(varab, int):
                    if varab in [0, 1]:
                        varab = bool(varab)
            self.pars.set(*li[0][1][-1], "\n".join(str(varab).splitlines()))
            # print(self.pars.__dict__)
        with open(self.cfgfile, "w") as configfile:
            self.pars.write(configfile)

    def _set_lables(self):
        co = 0
        for key_item, maxl in zip(self.cfgdictcopysorted.items(), self.maxlenlist):
            key, item = key_item
            la = str(f"label{co}")
            setattr(
                self,
                la,
                tk.Label(
                    self.subframe_in_canvas,
                    text=str(key),
                    font=self.mainlabelfont,
                    anchor="w",
                    justify=tk.LEFT,
                    width=self.maxlenabs,
                ),
            )
            getattr(getattr(self, la), "grid")(row=co, column=0, padx=5, pady=3)

            for ini, i in enumerate(item):
                subco = co + ini + 1
                lasub = str(f"sublabel{subco}")
                subcat = str(i[-1][-1][-1])
                valuetoadd = i[-1][0]
                setattr(
                    self,
                    lasub,
                    tk.Label(
                        self.subframe_in_canvas,
                        text=str(subcat),
                        font=self.sublabelfont,
                        anchor="w",
                        justify=tk.LEFT,
                        width=self.maxlenabs,
                    ),
                )
                getattr(getattr(self, lasub), "grid")(
                    row=subco, column=0, padx=1, pady=3
                )
                if not self.onezeroasboolean:
                    if isinstance_tolerant(valuetoadd, bool):
                        valuetoadd = int(valuetoadd)

                if isinstance_tolerant(valuetoadd, str):
                    valuetoadd = re.sub(r"\r", "", valuetoadd)
                    valuetoadd = re.sub(r"\n", "\\\\n", valuetoadd)
                    strvarname = f"stringvar{subco}"
                    inputtext = f"inputtext{subco}"

                    setattr(self, strvarname, tk.StringVar())
                    getattr(getattr(self, strvarname), "set")(valuetoadd)

                    setattr(
                        self,
                        inputtext,
                        tk.Entry(
                            self.subframe_in_canvas,
                            text=getattr(getattr(self, strvarname), "get")(),
                            textvariable=getattr(self, strvarname),
                            width=self.maxlenabsvals,
                            font=self.varfont,
                        ),
                    )
                    getattr(getattr(self, inputtext), "grid")(
                        row=subco, column=1, padx=1, pady=3
                    )

                    self.alladdedobjects[la][lasub][inputtext][strvarname] = valuetoadd
                    self.check_all_vars.append(
                        [
                            i,
                            la,
                            lasub,
                            inputtext,
                            strvarname,
                            valuetoadd,
                            getattr(getattr(self, strvarname), "get"),
                        ]
                    )

                elif isinstance_tolerant(valuetoadd, bool):

                    boolTrue = f"boolTrue{subco}"
                    boolFalse = f"boolFalse{subco}"
                    rbv = f"rbv{subco}"

                    setattr(self, rbv, tk.IntVar(self.subframe_in_canvas))
                    getattr(getattr(self, rbv), "set")(int(valuetoadd))

                    setattr(
                        self,
                        boolTrue,
                        tk.Radiobutton(
                            self.subframe_in_canvas,
                            text="True",
                            variable=getattr(self, rbv),
                            value=int(valuetoadd),
                            indicatoron=False,
                            command=lambda: (getattr(getattr(self, rbv), "get")()),
                            justify=tk.RIGHT,
                            anchor="w",
                            width=self.maxlenabsvals // 3,
                            font=self.varfont,
                        ),
                    )
                    setattr(
                        self,
                        boolFalse,
                        tk.Radiobutton(
                            self.subframe_in_canvas,
                            text="False",
                            variable=getattr(self, rbv),
                            value=int(not valuetoadd),
                            indicatoron=False,
                            command=lambda: (getattr(getattr(self, rbv), "get")()),
                            justify=tk.RIGHT,
                            anchor="w",
                            width=self.maxlenabsvals // 3,
                            font=self.varfont,
                        ),
                    )
                    getattr(getattr(self, boolTrue), "grid")(
                        row=subco, column=1, padx=1, pady=3
                    )
                    getattr(getattr(self, boolFalse), "grid")(
                        row=subco, column=2, padx=1, pady=3
                    )
                    self.check_all_vars.append(
                        [
                            i,
                            la,
                            lasub,
                            boolTrue,
                            rbv,
                            valuetoadd,
                            (getattr(getattr(self, rbv), "get")),
                        ]
                    )

                elif isinstance_tolerant(valuetoadd, int):

                    intvarname = f"intvar{subco}"
                    inputint = f"inputint{subco}"

                    setattr(self, intvarname, tk.IntVar())
                    getattr(getattr(self, intvarname), "set")(valuetoadd)

                    setattr(
                        self,
                        inputint,
                        tk.Entry(
                            self.subframe_in_canvas,
                            text=getattr(getattr(self, intvarname), "get")(),
                            textvariable=getattr(self, intvarname),
                            width=self.maxlenabsvals,
                            validate="all",
                            font=self.varfont,
                        ),
                    )

                    (
                        getattr(self, inputint).config(
                            validatecommand=(
                                getattr(self, inputint).register(on_validate_int),
                                "%P",
                            )
                        )
                    )

                    getattr(getattr(self, inputint), "grid")(
                        row=subco, column=1, padx=1, pady=3
                    )

                    self.alladdedobjects[la][lasub][inputint][intvarname] = valuetoadd
                    self.check_all_vars.append(
                        [
                            i,
                            la,
                            lasub,
                            inputint,
                            intvarname,
                            valuetoadd,
                            getattr(getattr(self, intvarname), "get"),
                        ]
                    )

                elif isinstance_tolerant(valuetoadd, float):

                    floatvarname = f"floatvar{subco}"
                    inputfloat = f"inputfloat{subco}"

                    setattr(self, floatvarname, tk.DoubleVar())
                    getattr(getattr(self, floatvarname), "set")(valuetoadd)

                    setattr(
                        self,
                        inputfloat,
                        tk.Entry(
                            self.subframe_in_canvas,
                            text=getattr(getattr(self, floatvarname), "get")(),
                            textvariable=getattr(self, floatvarname),
                            width=self.maxlenabsvals,
                            validate="all",
                            font=self.varfont,
                        ),
                    )

                    (
                        getattr(self, inputfloat).config(
                            validatecommand=(
                                getattr(self, inputfloat).register(on_validate_float),
                                "%P",
                            )
                        )
                    )

                    getattr(getattr(self, inputfloat), "grid")(
                        row=subco, column=1, padx=1, pady=3
                    )
                    self.check_all_vars.append(
                        [
                            i,
                            la,
                            lasub,
                            inputfloat,
                            floatvarname,
                            valuetoadd,
                            getattr(getattr(self, floatvarname), "get"),
                        ]
                    )

                    self.alladdedobjects[la][lasub][inputfloat][
                        floatvarname
                    ] = valuetoadd

                # print(self.alladdedobjects)

            co = co + maxl

    def _get_button_len(self):
        return self.maxlenabsvals//2 if (self.maxlenabsvals//2) > 15 else 15

    def mainloop(self):
        self.mainframe.pack(fill=tk.BOTH, expand=1)
        self.canvas_in_mainframe.pack(side=tk.LEFT, fill=tk.BOTH, expand=1)
        self.scrollbar_right_side.pack(side=tk.RIGHT, fill=tk.Y)
        self.canvas_in_mainframe.configure(yscrollcommand=self.scrollbar_right_side.set)
        self.canvas_in_mainframe.bind(
            "<Configure>",
            lambda e: self.canvas_in_mainframe.configure(
                scrollregion=self.canvas_in_mainframe.bbox("all")
            ),
        )
        self.canvas_in_mainframe.create_window(
            (0, 0), window=self.subframe_in_canvas, anchor="e"
        )
        self._set_lables()
        maxindex = max(
            [
                int(h)
                for h in list(
                    flatten_everything(
                        [
                            re.findall(r"\d+", str(q))
                            for q in flatten_everything(
                                [x[1] for x in (fla_tu(self.alladdedobjects))]
                            )
                        ]
                    )
                )
            ]
        )

        setattr(
            self,
            "button_save",
            tk.Button(
                self.subframe_in_canvas,
                text=str("Save").center(self.maxlenabsvals),
                command=lambda: self._update_cfg(),
                anchor="w",
                width=self._get_button_len(),
                font=self.buttonfont,
                justify=tk.RIGHT,
            ),
        )
        maxindex += 1
        getattr(getattr(self, "button_save"), "grid")(
            row=maxindex, column=0, padx=1, pady=3
        )

        setattr(
            self,
            "button_restore_default",
            tk.Button(
                self.subframe_in_canvas,
                text=str("Restore Default").center(self.maxlenabsvals),
                command=lambda: self._restore_cfg(),
                anchor="w",
                width=self._get_button_len(),
                font=self.buttonfont,
                justify=tk.RIGHT,
            ),
        )
        getattr(getattr(self, "button_restore_default"), "grid")(
            row=maxindex, column=1, padx=1, pady=3
        )
        setattr(
            self,
            "button_exit",
            tk.Button(
                self.subframe_in_canvas,
                text=str("Exit").center(self.maxlenabsvals),
                command=lambda: self.exit_app(),
                anchor="w",
                width=self._get_button_len(),
                font=self.buttonfont,
                justify=tk.RIGHT,
            ),
        )
        maxindex += 1
        getattr(getattr(self, "button_exit"), "grid")(
            row=maxindex, column=0, padx=1, pady=3
        )

        setattr(
            self,
            "button_exit_start_app",
            tk.Button(
                self.subframe_in_canvas,
                text=str("Start App").center(self.maxlenabsvals),
                command=lambda: self.exit_config_start_app(),
                anchor="w",
                width=self._get_button_len(),
                font=self.buttonfont,
                justify=tk.RIGHT,
            ),
        )
        getattr(getattr(self, "button_exit_start_app"), "grid")(
            row=maxindex, column=1, padx=1, pady=3
        )

        if self.addbuttons:
            for bu in self.addbuttons:
                maxindex += 1
                varabx = f"bu{maxindex}"
                setattr(
                    self,
                    varabx,
                    tk.Button(
                        self.subframe_in_canvas,
                        text=str(bu[0]).center(self.maxlenabsvals),
                        command=deepcopy(lambda: bu[1]()),
                        anchor="w", width=self._get_button_len(), font=self.buttonfont,
                        justify=tk.RIGHT,
                    ),
                )
                getattr(getattr(self, varabx), "grid")(
                    row=maxindex, column=0, padx=1, pady=3
                )

        self.canvas_in_mainframe.bind("<Enter>", self._bound_to_mousewheel)
        self.canvas_in_mainframe.bind("<Leave>", self._unbound_to_mousewheel)

        self.root.state("zoomed")
        #self.root.minsize(150, 100)
        self.root.mainloop()

    def _bound_to_mousewheel(self, event):
        self.canvas_in_mainframe.bind_all("<MouseWheel>", self._on_mousewheel)

    def _unbound_to_mousewheel(self, event):
        self.canvas_in_mainframe.unbind_all("<MouseWheel>")

    def _on_mousewheel(self, event):
        self.canvas_in_mainframe.yview_scroll(int(-1 * (event.delta / 120)), "units")


def start_config(
    cfgfile,
    title,
    icon=None,
    res="1024x768",
    addbuttons=(),
    mainlabelfont=("Helvetica", 15, "underline bold italic"),
    sublabelfont=("Helvetica", 14),
    varfont=("Helvetica", 10),
    buttonfont=("Helvetica", 12, "bold italic"),
):
    los = Cfedit(
        cfgfile=cfgfile,
        title=title,
        icon=icon,
        res=res,
        addbuttons=addbuttons,
        mainlabelfont=mainlabelfont,
        sublabelfont=sublabelfont,
        varfont=varfont,
        buttonfont=buttonfont,
    )
    los.mainloop()


def start_config_and_load_vars(
    cfgfile,
    title,
    icon=None,
    res="1024x768",
    addbuttons=(),
    mainlabelfont=("Helvetica", 15, "underline bold italic"),
    sublabelfont=("Helvetica", 14),
    varfont=("Helvetica", 10),
    buttonfont=("Helvetica", 12, "bold italic"),
    onezeroasboolean=True,
):
    los = Cfedit(
        cfgfile=cfgfile,
        title=title,
        icon=icon,
        res=res,
        addbuttons=addbuttons,
        mainlabelfont=mainlabelfont,
        sublabelfont=sublabelfont,
        varfont=varfont,
        buttonfont=buttonfont,
    )
    los.mainloop()
    load_config_file_vars(cfgfile, onezeroasboolean)
