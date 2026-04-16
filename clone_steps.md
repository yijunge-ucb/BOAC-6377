## Core logic extracted from the current code 

```python
# boac/api/degree_progress_student_controller.py
@app.route('/api/degree/check/batch', methods=['POST'])
@can_edit_degree_progress
def batch_degree_checks():
    params = request.get_json()
    sids = get_param(params, 'sids')
    template_id = get_param(params, 'templateId')
    if not template_id or not sids:
        raise BadRequestError('sids and templateId are required.')
    validate_template_not_archived(template_id)
    return tolerant_jsonify(create_batch_degree_checks(template_id=template_id, sids=sids))

# boac/api/degree_progress_api_utils.py
def create_batch_degree_checks(template_id, sids):
    benchmark = get_benchmarker(f'create_batch_degree_checks template_id={template_id}')
    benchmark('begin')
    template = fetch_degree_template(template_id)
    created_by = current_user.get_id()
    results_by_sid = {}
    benchmark(f'creating {len(sids)} clones')
    for sid in sids:
        degree_check = clone(template, created_by, sid=sid)
        results_by_sid[sid] = degree_check.id
    benchmark('end')
    return results_by_sid

# boac/api/degree_progress_api_utils.py
def clone(template, created_by, name=None, sid=None):
    template_clone = DegreeProgressTemplate.create(
        advisor_dept_codes=dept_codes_where_advising(current_user.departments),
        created_by=created_by,
        degree_name=name or template.degree_name,
        parent_template_id=template.id if sid else None,
        student_sid=sid,
    )
    unit_requirements_by_source_id = {}
    template_clone_id = template_clone.id
    for unit_requirement in template.unit_requirements:
        source_id = unit_requirement.id
        unit_requirements_by_source_id[source_id] = DegreeProgressUnitRequirement.create(
            created_by=created_by,
            min_units=unit_requirement.min_units,
            name=unit_requirement.name,
            template_id=template_clone_id,
        )

    categories = DegreeProgressCategory.get_categories(template_id=template.id)
    for category in categories:
        c = _clone_category(
            category=category,
            parent_id=None,
            sid=sid,
            template_id=template_clone_id,
            unit_requirements_by_source_id=unit_requirements_by_source_id,
        )
        for course in category['courseRequirements']:
            _clone_category(
                category=course,
                parent_id=c.id,
                sid=sid,
                template_id=template_clone_id,
                unit_requirements_by_source_id=unit_requirements_by_source_id,
            )
        for subcategory in category['subcategories']:
            s = _clone_category(
                category=subcategory,
                parent_id=c.id,
                sid=sid,
                template_id=template_clone_id,
                unit_requirements_by_source_id=unit_requirements_by_source_id,
            )
            for course in subcategory['courseRequirements']:
                _clone_category(
                    category=course,
                    parent_id=s.id,
                    sid=sid,
                    template_id=template_clone_id,
                    unit_requirements_by_source_id=unit_requirements_by_source_id,
                )
    return DegreeProgressTemplate.find_by_id(template_clone_id)

def _clone_category(
    category,
    parent_id,
    sid,
    template_id,
    unit_requirements_by_source_id,
):
    unit_requirement_ids = []
    for u in category['unitRequirements']:
        source_id_ = u['id']
        cross_reference = unit_requirements_by_source_id[source_id_]
        unit_requirement_ids.append(cross_reference.id)
    category_name = category['name']
    units_lower = category['unitsLower']
    units_upper = category['unitsUpper']
    is_satisfied_by_transfer_course = category['isSatisfiedByTransferCourse']
    now = datetime.now()

    category_created = DegreeProgressCategory.create(
        category_type=category['categoryType'],
        course_units_lower=units_lower,
        course_units_upper=units_upper,
        description=category['description'],
        is_satisfied_by_transfer_course=False if sid else is_satisfied_by_transfer_course,
        name=category_name,
        parent_category_id=parent_id,
        template_id=template_id,
        unit_requirement_ids=unit_requirement_ids,
        ux_position_x=category['uxPositionX'],
        ux_position_y=category['uxPositionY'],
    )
    if sid and is_satisfied_by_transfer_course:
        transfer_course = DegreeProgressCourse.create(
            accent_color='Purple',
            category_id=category_created.id,
            degree_check_id=template_id,
            display_name=category_name,
            grade='T',
            manually_created_at=now,
            manually_created_by=current_user.user_id,
            section_id=None,
            sid=sid,
            term_id=None,
            units=units_lower or units_upper,
            unit_requirement_ids=unit_requirement_ids,
        )
        DegreeProgressCourse.assign(
            category_id=category_created.id,
            course_id=transfer_course.id,
        )
    return category_created

class DegreeProgressTemplate(Base):
    __tablename__ = 'degree_progress_templates'

    id = db.Column(db.Integer, nullable=False, primary_key=True)
    advisor_dept_codes = db.Column(ARRAY(db.String), nullable=False)
    archived_at = db.Column(db.DateTime)
    created_by = db.Column(db.Integer, db.ForeignKey('authorized_users.id'), nullable=False)
    degree_name = db.Column(db.String(255), nullable=False)
    deleted_at = db.Column(db.DateTime, nullable=True)
    parent_template_id = db.Column(db.Integer, db.ForeignKey('degree_progress_templates.id'))
    student_sid = db.Column(db.String(80), nullable=True)
    updated_by = db.Column(db.Integer, db.ForeignKey('authorized_users.id'), nullable=False)

    note = db.relationship(
        DegreeProgressNote.__name__,
        back_populates='template',
        uselist=False,
    )
    unit_requirements = db.relationship(
        DegreeProgressUnitRequirement.__name__,
        back_populates='template',
        order_by='DegreeProgressUnitRequirement.created_at',
    )

    def __init__(self, advisor_dept_codes, created_by, degree_name, parent_template_id, student_sid, updated_by):
        self.advisor_dept_codes = advisor_dept_codes
        self.created_by = created_by
        self.degree_name = degree_name
        self.parent_template_id = parent_template_id
        self.student_sid = student_sid
        self.updated_by = updated_by

    @classmethod
    def create(
            cls,
            advisor_dept_codes,
            created_by,
            degree_name,
            parent_template_id=None,
            student_sid=None,
    ):
        degree = cls(
            advisor_dept_codes=advisor_dept_codes,
            created_by=created_by,
            degree_name=degree_name,
            parent_template_id=parent_template_id,
            student_sid=student_sid,
            updated_by=created_by,
        )
        db.session.add(degree)
        std_commit()
        return degree

    @classmethod
    def find_by_id(cls, template_id):
        return cls.query.filter_by(id=template_id, deleted_at=None).first()


class DegreeProgressUnitRequirement(Base):
    __tablename__ = 'degree_progress_unit_requirements'

    id = db.Column(db.Integer, nullable=False, primary_key=True)
    created_by = db.Column(db.Integer, db.ForeignKey('authorized_users.id'), nullable=False)
    min_units = db.Column(db.Integer, nullable=False)
    name = db.Column(db.String(255), nullable=False)
    template_id = db.Column(db.Integer, db.ForeignKey('degree_progress_templates.id'), nullable=False)
    updated_by = db.Column(db.Integer, db.ForeignKey('authorized_users.id'), nullable=False)
    categories = db.relationship(
        'DegreeProgressCategoryUnitRequirement',
        back_populates='unit_requirement',
    )
    courses = db.relationship(
        'DegreeProgressCourseUnitRequirement',
        back_populates='unit_requirement',
    )
    template = db.relationship('DegreeProgressTemplate', back_populates='unit_requirements')

    __table_args__ = (db.UniqueConstraint(
        'name',
        'template_id',
        name='degree_progress_unit_requirements_name_template_id_unique_const',
    ),)

    def __init__(self, created_by, min_units, name, template_id, updated_by):
        self.created_by = created_by
        self.min_units = min_units
        self.name = name
        self.template_id = template_id
        self.updated_by = updated_by

    @classmethod
    def create(cls, created_by, min_units, name, template_id):
        unit_requirement = cls(
            created_by=created_by,
            min_units=min_units,
            name=name,
            template_id=template_id,
            updated_by=created_by,
        )
        db.session.add(unit_requirement)
        std_commit()
        return unit_requirement

   
class DegreeProgressCategory(Base):
    __tablename__ = 'degree_progress_categories'

    id = db.Column(db.Integer, nullable=False, primary_key=True)
    accent_color = db.Column(db.String(255))
    category_type = db.Column(degree_progress_category_type, nullable=False)
    course_units = db.Column(NUMRANGE)
    description = db.Column(db.Text)
    grade = db.Column(db.String(50))
    is_ignored = db.Column(db.Boolean, nullable=False, default=False)
    is_recommended = db.Column(db.Boolean, nullable=False, default=False)
    is_satisfied_by_transfer_course = db.Column(db.Boolean, nullable=False, default=False)
    name = db.Column(db.String(255), nullable=False)
    note = db.Column(db.Text)
    parent_category_id = db.Column(db.Integer, db.ForeignKey('degree_progress_categories.id'))
    template_id = db.Column(db.Integer, db.ForeignKey('degree_progress_templates.id'), nullable=False)
    ux_position_x = db.Column(db.Integer, nullable=False)
    ux_position_y = db.Column(db.Integer, nullable=False)
    unit_requirements = db.relationship(
        DegreeProgressCategoryUnitRequirement.__name__,
        back_populates='category',
        lazy='joined',
    )

    def __init__(
            self,
            category_type,
            name,
            template_id,
            ux_position_x,
            ux_position_y,
            accent_color=None,
            course_units=None,
            description=None,
            grade=None,
            is_ignored=False,
            is_recommended=False,
            is_satisfied_by_transfer_course=False,
            parent_category_id=None,
    ):
        self.accent_color = accent_color
        self.category_type = category_type
        self.course_units = course_units
        self.description = description
        self.grade = grade
        self.is_ignored = is_ignored
        self.is_recommended = is_recommended
        self.is_satisfied_by_transfer_course = is_satisfied_by_transfer_course
        self.name = name
        self.parent_category_id = parent_category_id
        self.template_id = template_id
        self.ux_position_x = ux_position_x
        self.ux_position_y = ux_position_y

    @classmethod
    def create(
            cls,
            category_type,
            name,
            template_id,
            ux_position_x,
            accent_color=None,
            course_units_lower=None,
            course_units_upper=None,
            description=None,
            grade=None,
            is_satisfied_by_transfer_course=False,
            parent_category_id=None,
            unit_requirement_ids=None,
            ux_position_y=None,
    ):
        course_units = None if course_units_lower is None else NumericRange(
            float(course_units_lower),
            float(course_units_upper or course_units_lower),
            '[]',
        )
        if not ux_position_y:
            # Auto-calculate the 'ux_position_y' value.
            list_of_ux_position_y = cls.fetch_list_of_ux_position_y(
                category_type=category_type,
                parent_category_id=parent_category_id,
                template_id=template_id,
                ux_position_x=ux_position_x,
            )
            ux_position_y = min(list_of_ux_position_y) - 1 if len(list_of_ux_position_y) else 0

        category = cls(
            accent_color=accent_color,
            category_type=category_type,
            course_units=course_units,
            description=description,
            grade=grade,
            is_satisfied_by_transfer_course=is_satisfied_by_transfer_course,
            name=name,
            parent_category_id=parent_category_id,
            template_id=template_id,
            ux_position_x=ux_position_x,
            ux_position_y=ux_position_y,
        )
        # TODO: Use 'unit_requirement_ids' in mapping this instance to 'unit_requirements' table
        db.session.add(category)
        std_commit()
        for unit_requirement_id in unit_requirement_ids or []:
            DegreeProgressCategoryUnitRequirement.create(
                category_id=category.id,
                unit_requirement_id=int(unit_requirement_id),
            )
        return category

    @classmethod
    def get_categories(cls, template_id):
        hierarchy = []
        categories = []
        for category in cls.query.filter_by(template_id=template_id).all():
            category_type = category.category_type
            api_json = category.to_api_json()
            if category_type == 'Category':
                # A 'Category' can have both courses and subcategories. A 'Subcategory' can have courses.
                api_json['courseRequirements'] = []
                api_json['subcategories'] = []
            elif category_type == 'Subcategory':
                api_json['courseRequirements'] = []
            categories.append(api_json)

        categories_by_id = dict((category['id'], category) for category in categories)
        for category in categories:
            parent_category_id = category['parentCategoryId']
            if parent_category_id:
                parent = categories_by_id[parent_category_id]
                key = 'subcategories' if category['categoryType'] == 'Subcategory' else 'courseRequirements'
                parent[key].append(category)
            else:
                hierarchy.append(category)

        def _sort(_categories):
            # Order by ux_position_y, descending.
            _categories = sorted(_categories, key=lambda c: c['uxPositionY'], reverse=True)
            for _category in _categories:
                # Order courses and course_requirements by name, ascending.
                _category['courses'] = sorted(_category['courses'], key=lambda c: c['createdAt'])
                _category['courseRequirements'] = sorted(_category['courseRequirements'], key=lambda c: c['createdAt'])
            return _categories
        hierarchy = _sort(hierarchy)
        for category in hierarchy:
            category['subcategories'] = _sort(category['subcategories'])
        return hierarchy

class DegreeProgressCategoryUnitRequirement(db.Model):
    __tablename__ = 'degree_progress_category_unit_requirements'

    category_id = db.Column(db.Integer, db.ForeignKey('degree_progress_categories.id'), primary_key=True)
    unit_requirement_id = db.Column(db.Integer, db.ForeignKey('degree_progress_unit_requirements.id'), primary_key=True)
    category = db.relationship('DegreeProgressCategory', back_populates='unit_requirements')
    unit_requirement = db.relationship('DegreeProgressUnitRequirement', back_populates='categories')

    def __init__(
            self,
            category_id,
            unit_requirement_id,
    ):
        self.category_id = category_id
        self.unit_requirement_id = unit_requirement_id

    @classmethod
    def create(cls, category_id, unit_requirement_id):
        db.session.add(cls(category_id=category_id, unit_requirement_id=unit_requirement_id))
        std_commit()

    @classmethod
    def find_by_category_id(cls, category_id):
        return cls.query.filter_by(category_id=category_id).all()

class DegreeProgressCourse(Base):
    __tablename__ = 'degree_progress_courses'

    id = db.Column(db.Integer, nullable=False, primary_key=True)
    accent_color = db.Column(db.String(255))
    category_id = db.Column(db.Integer, db.ForeignKey('degree_progress_categories.id'))
    degree_check_id = db.Column(db.Integer, db.ForeignKey('degree_progress_templates.id'), nullable=False)
    display_name = db.Column(db.String(255), nullable=False)
    grade = db.Column(db.String(50), nullable=False)
    ignore = db.Column(db.Boolean, nullable=False)
    note = db.Column(db.Text)
    manually_created_at = db.Column(db.DateTime)
    manually_created_by = db.Column(db.Integer, db.ForeignKey('authorized_users.id'))
    section_id = db.Column(db.Integer)
    sid = db.Column(db.String(80), nullable=False)
    term_id = db.Column(db.Integer)
    units = db.Column(db.Numeric, nullable=False)
    unit_requirements = db.relationship(
        DegreeProgressCourseUnitRequirement.__name__,
        back_populates='course',
        lazy='joined',
    )

    __table_args__ = (db.UniqueConstraint(
        'category_id',
        'degree_check_id',
        'manually_created_at',
        'manually_created_by',
        'section_id',
        'sid',
        'term_id',
        name='degree_progress_courses_category_id_course_unique_constraint',
    ),)

    def __init__(
            self,
            degree_check_id,
            display_name,
            grade,
            section_id,
            sid,
            term_id,
            units,
            accent_color=None,
            category_id=None,
            ignore=False,
            manually_created_at=None,
            manually_created_by=None,
            note=None,
    ):
        self.accent_color = accent_color
        self.category_id = category_id
        self.degree_check_id = degree_check_id
        self.display_name = display_name
        self.grade = grade
        self.ignore = ignore
        self.manually_created_by = manually_created_by
        if self.manually_created_by and not manually_created_at:
            raise ValueError('manually_created_at is required if manually_created_by is present.')
        else:
            self.manually_created_at = manually_created_at
        self.note = note.strip() if note else None
        self.section_id = section_id
        self.sid = sid
        self.term_id = term_id
        self.units = units

    @classmethod
    def assign(cls, category_id, course_id):
        course = cls.query.filter_by(id=course_id).first()
        course.category_id = category_id
        course.ignore = False
        std_commit()
        DegreeProgressCourseUnitRequirement.delete(course_id)
        for u in DegreeProgressCategoryUnitRequirement.find_by_category_id(category_id):
            DegreeProgressCourseUnitRequirement.create(course.id, u.unit_requirement_id)
        return course

    @classmethod
    def create(
            cls,
            degree_check_id,
            display_name,
            grade,
            section_id,
            sid,
            term_id,
            units,
            accent_color=None,
            category_id=None,
            manually_created_at=None,
            manually_created_by=None,
            note=None,
            unit_requirement_ids=(),
    ):
        course = cls(
            accent_color=accent_color,
            category_id=category_id,
            degree_check_id=degree_check_id,
            display_name=display_name,
            grade=grade,
            manually_created_at=manually_created_at,
            manually_created_by=manually_created_by,
            note=note,
            section_id=section_id,
            sid=sid,
            term_id=term_id,
            units=units if (units is None or is_float(units)) else 0,
        )
        db.session.add(course)
        std_commit()

        for unit_requirement_id in unit_requirement_ids:
            DegreeProgressCourseUnitRequirement.create(
                course_id=course.id,
                unit_requirement_id=unit_requirement_id,
            )
        return course


class DegreeProgressCourseUnitRequirement(db.Model):
    __tablename__ = 'degree_progress_course_unit_requirements'

    course_id = db.Column(db.Integer, db.ForeignKey('degree_progress_courses.id'), primary_key=True)
    unit_requirement_id = db.Column(db.Integer, db.ForeignKey('degree_progress_unit_requirements.id'), primary_key=True)
    course = db.relationship('DegreeProgressCourse', back_populates='unit_requirements')
    unit_requirement = db.relationship('DegreeProgressUnitRequirement', back_populates='courses')

    def __init__(
            self,
            course_id,
            unit_requirement_id,
    ):
        self.course_id = course_id
        self.unit_requirement_id = unit_requirement_id

    @classmethod
    def create(cls, course_id, unit_requirement_id):
        db.session.add(cls(course_id=course_id, unit_requirement_id=unit_requirement_id))
        std_commit()

    @classmethod
    def delete(cls, course_id):
        for row in cls.find_by_course_id(course_id):
            db.session.delete(row)
        std_commit()

```
## Example of cloning a degree check template with template_id=100 to a new template with parent_template_id=100 and student_sid=12345, and the related database operations:

### 1. Scan the table degree_progress_templates for template with id=100

```sql
SELECT *
FROM degree_progress_templates
WHERE id = 100 AND deleted_at IS NULL;
```

#### degree_progress_templates

|id | advisor_dept_codes | archived_at | created_by | degree_name  | deleted_at | parent_template_id | student_sid | updated_by |
|---|--------------------|-------------|------------|--------------|------------|--------------------|-------------|------------|
|100| {"CS","EECS"}      | NULL        | 42         | CS Template  | NULL       | NULL                | NULL        | 42        |


### 2. Insert a new row into the table degree_progress_templates with parent_template_id=100 and student_sid=12345, and then commit the transaction.

```sql
INSERT INTO degree_progress_templates
(advisor_dept_codes, created_by, degree_name, parent_template_id, student_sid, updated_by)
VALUES
('{"CS","EECS"}', 42, 'CS Template', 100, '12345', 42)
RETURNING id;

COMMIT;
```

#### degree_progress_templates

|id | advisor_dept_codes | archived_at | created_by | degree_name  | deleted_at | parent_template_id | student_sid | updated_by|
|---|---------------------|-------------|------------|--------------|------------|---------------------|-------------|-----------|
|100| {"CS","EECS"}      | NULL        | 42         | CS Template  | NULL       | NULL                | NULL        | 42         |
|200| {"CS","EECS"}      | NULL        | 42         | CS Template  | NULL       | 100                 | 12345       | 42         |


### 3. Scan the table degree_progress_unit_requirements for rows with template_id=100.

```sql
SELECT *
FROM degree_progress_unit_requirements
WHERE template_id = 100;
```
#### degree_progress_unit_requirements

id | created_by | min_units | name        | template_id | updated_by
---|------------|-----------|-------------|-------------|-----------
1  | 42         | 10        | Math Units  | 100         | 42
2  | 42         | 20        | CS Units    | 100         | 42

### 4. Entering a for loop for cloning unit requirements for template_id=100 to template_id=200.
    
#### 4.1. Insert a new row into the table degree_progress_unit_requirements with template_id=200 for unit requirement with id=1, and then commit the transaction.

```sql
INSERT INTO degree_progress_unit_requirements
(created_by, min_units, name, template_id, updated_by)
VALUES (42, 10, 'Math Units', 200, 42)
RETURNING id;   -- → 10

COMMIT;
```
#### degree_progress_unit_requirements
id | created_by | min_units | name        | template_id | updated_by
---|------------|-----------|-------------|-------------|-----------
1  | 42         | 10        | Math Units  | 100         | 42
2  | 42         | 20        | CS Units    | 100         | 42
10 | 42         | 10        | Math Units  | 200         | 42

#### 4.2. Insert a new row into the table degree_progress_unit_requirements with template_id=200 for unit requirement with id=2, and then commit the transaction.

```sql
INSERT INTO degree_progress_unit_requirements
(created_by, min_units, name, template_id, updated_by)
VALUES (42, 20, 'CS Units', 200, 42)
RETURNING id;   -- → 11

COMMIT;
```
#### degree_progress_unit_requirements
id | created_by | min_units | name        | template_id | updated_by
---|------------|-----------|-------------|-------------|-----------
1  | 42         | 10        | Math Units  | 100         | 42
2  | 42         | 20        | CS Units    | 100         | 42
10 | 42         | 10        | Math Units  | 200         | 42
11 | 42         | 20        | CS Units    | 200         | 42

### 5. Scan the table degree_progress_categories for rows with template_id=100 to construct the category trees.

#### degree_progress_categories
id   | name           | parent_category_id | template_id
-----|----------------|--------------------|-------------
1000 | Category A     | NULL               | 100
1001 | Course C1      | 1000               | 100
1002 | Course C2      | 1000               | 100
1003 | Subcategory B  | 1000               | 100
1004 | Course C3      | 1003               | 100
1005 | Course C4      | 1003               | 100


### 6. Entering a for loop to clone categories, courses under categories, subcategories under categories, and courses under subcategories for template_id=100 to template_id=200.
    
### 6.1. Clone the category with id=1000.
#### 6.1.1. Update the table degree_progress_categories with the new row for the cloned category, and then commit the transaction.

```sql
INSERT INTO degree_progress_categories
{name, parent_category_id, template_id}
VALUES ('Category A', NULL, 200) RETURNING id;   -- → 2000

COMMIT;
```

#### degree_progress_categories
id   | name           | parent_category_id | template_id
-----|----------------|--------------------|-------------
1000 | Category A     | NULL               | 100
1001 | Course C1      | 1000               | 100
1002 | Course C2      | 1000               | 100
1003 | Subcategory B  | 1000               | 100
1004 | Course C3      | 1003               | 100
1005 | Course C4      | 1003               | 100
2000 | Category A     | NULL               | 200

#### 6.1.2. Entering a for loop to clone unit requirements for this category.
##### 6.1.2.1. Update the table degree_progress_category_unit_requirements with the new row for the cloned category and unit requirement with id=10, and then commit the transaction.

```sql
INSERT INTO degree_progress_category_unit_requirements (2000, 10)

COMMIT;
```

#### degree_progress_category_unit_requirements
category_id | unit_requirement_id
------------|---------------------
1000        | 1
1000        | 2
1001        | 1
1002        | 1
1003        | 2
1004        | 2
1005        | 2
2000        | 10

##### 6.1.2.2. Update the table degree_progress_category_unit_requirements with the new row for the cloned category and unit requirement with id=11, and then commit the transaction.

#### degree_progress_category_unit_requirements
category_id | unit_requirement_id
------------|---------------------
1000        | 1
1000        | 2
1001        | 1
1002        | 1
1003        | 2
1004        | 2
1005        | 2
2000        | 10
2000        | 11

If this category can be satisfied by transfer courses, update the table degree_progress_courses and degree_progress_course_unit_requirements.

#### 6.1.3. Entering a for loop for cloning courses under the category with parent_category_id=1000.
##### 6.1.3.1. Clone Course C1. 

Update the table degree_progress_categories with the new row for the cloned course with parent_category_id=2000, and then commit the transaction.

```sql
INSERT category (Course C1, parent=2000)
```

#### degree_progress_categories
id   | name           | parent_category_id | template_id
-----|----------------|--------------------|-------------
1000 | Category A     | NULL               | 100
1001 | Course C1      | 1000               | 100
1002 | Course C2      | 1000               | 100
1003 | Subcategory B  | 1000               | 100
1004 | Course C3      | 1003               | 100
1005 | Course C4      | 1003               | 100
2000 | Category A     | NULL               | 200
2001 | Course C1      | 2000               | 200
            
Entering a for loop to update the table degree_progress_category_unit_requirements for the cloned course with parent_category_id=2000.
Update the table degree_progress_category_unit_requirements with the new row for the cloned course and unit requirement with id=10, and then commit the transaction.

```sql
INSERT cat_unit_req (2001,10) + COMMIT
```

#### degree_progress_category_unit_requirements
category_id | unit_requirement_id
------------|---------------------
1000        | 1
1000        | 2
1001        | 1
1002        | 1
1003        | 2
1004        | 2
1005        | 2
2000        | 10
2000        | 11
2001        | 10

If this category can be satisfied by transfer courses, update the table degree_progress_courses and degree_progress_course_unit_requirements.

##### 6.1.3.2. Clone Course C2. 

Update the table degree_progress_categories with the new row for the cloned course with parent_category_id=2000, and then commit the transaction.

```sql
INSERT category (Course C2, parent=2000)
```

#### degree_progress_categories
id   | name           | parent_category_id | template_id
-----|----------------|--------------------|-------------
1000 | Category A     | NULL               | 100
1001 | Course C1      | 1000               | 100
1002 | Course C2      | 1000               | 100
1003 | Subcategory B  | 1000               | 100
1004 | Course C3      | 1003               | 100
1005 | Course C4      | 1003               | 100
2000 | Category A     | NULL               | 200
2001 | Course C1      | 2000               | 200
2002 | Course C2      | 2000               | 200

Entering a for loop to update the table degree_progress_category_unit_requirements for the cloned course with parent_category_id=2000.
Update the table degree_progress_category_unit_requirements with the new row for the cloned course and unit requirement with id=10, and then commit the transaction.

```sql
INSERT cat_unit_req (2002,10) + COMMIT
```

#### degree_progress_category_unit_requirements
category_id | unit_requirement_id
------------|---------------------
1000        | 1
1000        | 2
1001        | 1
1002        | 1
1003        | 2
1004        | 2
1005        | 2
2000        | 10
2000        | 11
2001        | 10
2002        | 10

If this category can be satisfied by transfer courses, update the table degree_progress_courses and degree_progress_course_unit_requirements.
        
#### 6.1.4. Enter a for loop to clone subcategories
##### 6.1.4.1. Update the table degree_progress_categories with the new row for the cloned subcategory with parent_category_id=2000, and then commit the transaction.

```sql
INSERT Subcategory B (parent=2000)
```
#### degree_progress_categories
id   | name           | parent_category_id | template_id
-----|----------------|--------------------|-------------
1000 | Category A     | NULL               | 100
1001 | Course C1      | 1000               | 100
1002 | Course C2      | 1000               | 100
1003 | Subcategory B  | 1000               | 100
1004 | Course C3      | 1003               | 100
1005 | Course C4      | 1003               | 100
2000 | Category A     | NULL               | 200
2001 | Course C1      | 2000               | 200
2002 | Course C2      | 2000               | 200
2003 | Subcategory B  | 2000               | 200

##### 6.1.4.2. Entering a for loop to update the table degree_progress_category_unit_requirements for the cloned subcategory with parent_category_id=2000.
###### 6.1.4.2.1. Update the table degree_progress_category_unit_requirements with the new row for the cloned subcategory and unit requirement with id=11, and then commit the transaction.

```sql                    
INSERT cat_unit_req (2003,11) + COMMIT
```

#### degree_progress_category_unit_requirements
category_id | unit_requirement_id
------------|---------------------
1000        | 1
1000        | 2
1001        | 1
1002        | 1
1003        | 2
1004        | 2
1005        | 2
2000        | 10
2000        | 11
2001        | 10
2002        | 10
2003        | 11

If this category can be satisfied by transfer courses, update the table degree_progress_courses and degree_progress_course_unit_requirements.
            
##### 6.1.4.3. Enter a for loop to clone courses under subcategories. 
###### 6.1.4.3.1. Clone Course C3.
Update the table degree_progress_categories with the new row for the cloned course with parent_category_id=2003, and then commit the transaction.

```sql
Insert course (Course C3, parent=2003) + COMMIT
```
#### degree_progress_categories
id   | name           | parent_category_id | template_id
-----|----------------|--------------------|-------------
1000 | Category A     | NULL               | 100
1001 | Course C1      | 1000               | 100
1002 | Course C2      | 1000               | 100
1003 | Subcategory B  | 1000               | 100
1004 | Course C3      | 1003               | 100
1005 | Course C4      | 1003               | 100
2000 | Category A     | NULL               | 200
2001 | Course C1      | 2000               | 200
2002 | Course C2      | 2000               | 200
2003 | Subcategory B  | 2000               | 200
2004 | Course C3      | 2003               | 200
                
Enter a for loop to update unit requirements. 
Update the table degree_progress_category_unit_requirements with the new row for the cloned course with parent_category_id=2003, and then commit the transaction.

```sql
INSERT cat_unit_req (2004,11) + COMMIT
```
#### degree_progress_category_unit_requirements
category_id | unit_requirement_id
------------|---------------------
1000        | 1
1000        | 2
1001        | 1
1002        | 1
1003        | 2
1004        | 2
1005        | 2
2000        | 10
2000        | 11
2001        | 10
2002        | 10
2003        | 11
2004        | 11

If this category can be satisfied by transfer courses, update the table degree_progress_courses and degree_progress_course_unit_requirements.

###### 6.1.4.3.2. Clone Course C4. 
Update the table degree_progress_categories with the new row for the cloned course with parent_category_id=2003, and then commit the transaction.

```sql
Insert course (Course C4, parent=2003) + COMMIT
```

#### degree_progress_categories
id   | name           | parent_category_id | template_id
-----|----------------|--------------------|-------------
1000 | Category A     | NULL               | 100
1001 | Course C1      | 1000               | 100
1002 | Course C2      | 1000               | 100
1003 | Subcategory B  | 1000               | 100
1004 | Course C3      | 1003               | 100
1005 | Course C4      | 1003               | 100
2000 | Category A     | NULL               | 200
2001 | Course C1      | 2000               | 200
2002 | Course C2      | 2000               | 200
2003 | Subcategory B  | 2000               | 200
2004 | Course C3      | 2003               | 200
2005 | Course C4      | 2003               | 200

Entering a for loop to update unit requirements. 
Update the table degree_progress_category_unit_requirements with the new row for the cloned course with parent_category_id=2003, and then commit the transaction.

```sql        
INSERT cat_unit_req (2005,11) + COMMIT
```

#### degree_progress_category_unit_requirements
category_id | unit_requirement_id
------------|---------------------
1000        | 1
1000        | 2
1001        | 1
1002        | 1
1003        | 2
1004        | 2
1005        | 2
2000        | 10
2000        | 11
2001        | 10
2002        | 10
2003        | 11
2004        | 11
2005        | 11

If this category can be satisfied by transfer courses, update the table degree_progress_courses and degree_progress_course_unit_requirements.


## Process of cloning transfer courses:

#### 1. INSERT course (3000) + COMMIT

#### degree_progress_courses
| id | accent_color | category_id | degree_check_id | display_name | grade | ignore | sid   | units |
-----|-------------- |------------|-----------------|--------------|-------|--------|-------|-------|
| 3000 | Purple       | 2000        | 200             | T Course     | T     | false| 12345 | 2 |


#### 2. DELETE course_unit_req + COMMIT

```sql
DELETE FROM degree_progress_course_unit_requirements
WHERE course_id = 3000;

COMMIT;
```

#### 3. SELECT cat_unit_req
```sql
SELECT unit_requirement_id
FROM degree_progress_category_unit_requirements
WHERE category_id = 2000;
```

#### 4. INSERT course_unit_req (3000,10) + COMMIT
```sql
INSERT INTO degree_progress_course_unit_requirements
(course_id=3000, unit_requirement_id=10);

COMMIT;
```
#### degree_progress_course_unit_requirements
category_id | unit_requirement_id
------------|---------------------
3000        | 10

#### 5. INSERT course_unit_req (3000,11) + COMMIT

```sql
INSERT INTO degree_progress_course_unit_requirements
(course_id=3000, unit_requirement_id=11);
```
#### degree_progress_course_unit_requirements
category_id | unit_requirement_id
------------|---------------------
3000        | 10
3000        | 11

